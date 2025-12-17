---
title: FastAPI Dapr Agent
description: Deploy FastAPI service with Dapr sidecar, Kafka pub/sub, and PostgreSQL integration
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - helm
  - docker
---

# FastAPI Dapr Agent

## Goal
Deploy a FastAPI microservice with Dapr sidecar to Kubernetes, featuring:
- Dapr service invocation and pub/sub
- Kafka integration for event-driven messaging
- PostgreSQL database connection
- Health and readiness probes
- Complete observability and validation

This service serves as a reference implementation for cloud-native Python agents in Hackathon III.

## Prerequisites Check
Before proceeding, verify:
1. Minikube is running with Dapr installed
2. kubectl is configured and accessible
3. Helm 3.x is installed
4. Docker is installed for building images
5. Kafka is deployed (use kafka-k8s-setup playbook)
6. PostgreSQL is deployed (use postgres-k8s-setup playbook)

## Instructions

### 1. Verify Prerequisites

```bash
# Check Minikube status
minikube status

# Verify kubectl access
kubectl cluster-info
kubectl get nodes

# Check Helm version (must be 3.x+)
helm version --short

# Verify Docker
docker --version

# Check Dapr installation
dapr --version
kubectl get pods -n dapr-system

# Verify Kafka is running
kubectl get pods -n kafka

# Verify PostgreSQL is running
kubectl get pods -n postgres
```

### 2. Install Dapr on Kubernetes (if not already installed)

```bash
# Initialize Dapr on Kubernetes cluster
dapr init -k

# Wait for Dapr system pods to be ready
kubectl wait --for=condition=Ready pod -l app=dapr-operator -n dapr-system --timeout=300s
kubectl wait --for=condition=Ready pod -l app=dapr-sidecar-injector -n dapr-system --timeout=300s
kubectl wait --for=condition=Ready pod -l app=dapr-sentry -n dapr-system --timeout=300s

# Verify Dapr installation
kubectl get pods -n dapr-system
```

### 3. Create Application Namespace

```bash
# Create namespace for the FastAPI service
kubectl create namespace fastapi-app --dry-run=client -o yaml | kubectl apply -f -

# Label namespace for automatic Dapr sidecar injection
kubectl label namespace fastapi-app dapr.io/enabled=true
```

### 4. Create Dapr Components

#### 4.1 Kafka Pub/Sub Component

```bash
# Create Dapr Kafka pubsub component
cat <<EOF | kubectl apply -f -
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-pubsub
  namespace: fastapi-app
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: "kafka.kafka.svc.cluster.local:9092"
  - name: authType
    value: "none"
  - name: consumerGroup
    value: "fastapi-consumer-group"
  - name: clientID
    value: "fastapi-dapr-agent"
EOF
```

#### 4.2 PostgreSQL State Store Component

```bash
# Get PostgreSQL password from secret
POSTGRES_PASSWORD=$(kubectl get secret -n postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode)

# Create Dapr PostgreSQL state store component
cat <<EOF | kubectl apply -f -
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: postgres-statestore
  namespace: fastapi-app
spec:
  type: state.postgresql
  version: v1
  metadata:
  - name: connectionString
    value: "host=postgres-postgresql.postgres.svc.cluster.local user=postgres password=${POSTGRES_PASSWORD} port=5432 database=postgres sslmode=disable"
  - name: tableName
    value: "dapr_state"
  - name: keyLength
    value: "255"
EOF
```

### 5. Create FastAPI Application

#### 5.1 Create Application Directory Structure

```bash
# Create project directory
mkdir -p /tmp/fastapi-dapr-agent
cd /tmp/fastapi-dapr-agent

# Create application structure
mkdir -p app
```

#### 5.2 Create FastAPI Application Code

```bash
# Create main application file
cat <<'EOF' > app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import httpx
import os
import logging
from typing import Dict, Any, Optional
import asyncpg

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="FastAPI Dapr Agent", version="1.0.0")

# Configuration
DAPR_HTTP_PORT = os.getenv("DAPR_HTTP_PORT", "3500")
DAPR_GRPC_PORT = os.getenv("DAPR_GRPC_PORT", "50001")
DAPR_URL = f"http://localhost:{DAPR_HTTP_PORT}"
PUBSUB_NAME = "kafka-pubsub"
TOPIC_NAME = "agent-events"
STATE_STORE_NAME = "postgres-statestore"

# PostgreSQL connection details
PG_HOST = os.getenv("POSTGRES_HOST", "postgres-postgresql.postgres.svc.cluster.local")
PG_PORT = os.getenv("POSTGRES_PORT", "5432")
PG_USER = os.getenv("POSTGRES_USER", "postgres")
PG_PASSWORD = os.getenv("POSTGRES_PASSWORD", "postgres")
PG_DATABASE = os.getenv("POSTGRES_DATABASE", "postgres")

# Database connection pool
db_pool: Optional[asyncpg.Pool] = None

# Models
class EventData(BaseModel):
    message: str
    metadata: Dict[str, Any] = {}

class StateData(BaseModel):
    key: str
    value: Dict[str, Any]

# Startup event
@app.on_event("startup")
async def startup_event():
    global db_pool
    try:
        # Create PostgreSQL connection pool
        db_pool = await asyncpg.create_pool(
            host=PG_HOST,
            port=int(PG_PORT),
            user=PG_USER,
            password=PG_PASSWORD,
            database=PG_DATABASE,
            min_size=2,
            max_size=10
        )
        logger.info("PostgreSQL connection pool created successfully")

        # Test database connection
        async with db_pool.acquire() as conn:
            version = await conn.fetchval("SELECT version()")
            logger.info(f"Connected to PostgreSQL: {version}")
    except Exception as e:
        logger.error(f"Failed to connect to PostgreSQL: {e}")

# Shutdown event
@app.on_event("shutdown")
async def shutdown_event():
    global db_pool
    if db_pool:
        await db_pool.close()
        logger.info("PostgreSQL connection pool closed")

# Health endpoints
@app.get("/health")
async def health():
    """Health check endpoint"""
    return {"status": "healthy", "service": "fastapi-dapr-agent"}

@app.get("/ready")
async def readiness():
    """Readiness check endpoint"""
    checks = {
        "api": "ok",
        "database": "unknown",
        "dapr": "unknown"
    }

    # Check database connection
    try:
        if db_pool:
            async with db_pool.acquire() as conn:
                await conn.fetchval("SELECT 1")
            checks["database"] = "ok"
        else:
            checks["database"] = "not_initialized"
    except Exception as e:
        checks["database"] = f"error: {str(e)}"

    # Check Dapr sidecar
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(f"{DAPR_URL}/v1.0/healthz")
            if response.status_code == 204:
                checks["dapr"] = "ok"
            else:
                checks["dapr"] = f"status_code: {response.status_code}"
    except Exception as e:
        checks["dapr"] = f"error: {str(e)}"

    # Overall status
    all_ok = all(v == "ok" for v in checks.values())
    status_code = 200 if all_ok else 503

    return checks if status_code == 200 else HTTPException(status_code=status_code, detail=checks)

# Dapr Pub/Sub endpoints
@app.post("/publish")
async def publish_event(event: EventData):
    """Publish event to Kafka via Dapr"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{DAPR_URL}/v1.0/publish/{PUBSUB_NAME}/{TOPIC_NAME}",
                json=event.dict()
            )
            response.raise_for_status()
            logger.info(f"Published event to topic {TOPIC_NAME}: {event.message}")
            return {"status": "published", "topic": TOPIC_NAME, "message": event.message}
    except Exception as e:
        logger.error(f"Failed to publish event: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/subscribe")
async def subscribe_handler(event: Dict[str, Any]):
    """Handle subscribed events from Kafka via Dapr"""
    try:
        logger.info(f"Received event from Kafka: {event}")
        # Process the event (placeholder for business logic)
        return {"status": "processed", "event": event}
    except Exception as e:
        logger.error(f"Failed to process event: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/dapr/subscribe")
async def dapr_subscribe():
    """Dapr subscription endpoint"""
    return [
        {
            "pubsubname": PUBSUB_NAME,
            "topic": TOPIC_NAME,
            "route": "/subscribe"
        }
    ]

# Dapr State Management endpoints
@app.post("/state")
async def save_state(state: StateData):
    """Save state to PostgreSQL via Dapr"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{DAPR_URL}/v1.0/state/{STATE_STORE_NAME}",
                json=[
                    {
                        "key": state.key,
                        "value": state.value
                    }
                ]
            )
            response.raise_for_status()
            logger.info(f"Saved state with key: {state.key}")
            return {"status": "saved", "key": state.key}
    except Exception as e:
        logger.error(f"Failed to save state: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/state/{key}")
async def get_state(key: str):
    """Get state from PostgreSQL via Dapr"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{DAPR_URL}/v1.0/state/{STATE_STORE_NAME}/{key}"
            )
            if response.status_code == 204:
                raise HTTPException(status_code=404, detail="State not found")
            response.raise_for_status()
            logger.info(f"Retrieved state with key: {key}")
            return {"key": key, "value": response.json()}
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Failed to get state: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/state/{key}")
async def delete_state(key: str):
    """Delete state from PostgreSQL via Dapr"""
    try:
        async with httpx.AsyncClient() as client:
            response = await client.delete(
                f"{DAPR_URL}/v1.0/state/{STATE_STORE_NAME}/{key}"
            )
            response.raise_for_status()
            logger.info(f"Deleted state with key: {key}")
            return {"status": "deleted", "key": key}
    except Exception as e:
        logger.error(f"Failed to delete state: {e}")
        raise HTTPException(status_code=500, detail=str(e))

# Direct PostgreSQL query endpoint (for validation)
@app.get("/db/query")
async def execute_query():
    """Execute test query directly on PostgreSQL"""
    try:
        if not db_pool:
            raise HTTPException(status_code=503, detail="Database pool not initialized")

        async with db_pool.acquire() as conn:
            # Get database version
            version = await conn.fetchval("SELECT version()")

            # Count tables
            table_count = await conn.fetchval("""
                SELECT COUNT(*)
                FROM information_schema.tables
                WHERE table_schema = 'public'
            """)

            return {
                "status": "ok",
                "postgres_version": version.split(",")[0],
                "table_count": table_count
            }
    except Exception as e:
        logger.error(f"Database query failed: {e}")
        raise HTTPException(status_code=500, detail=str(e))

# Root endpoint
@app.get("/")
async def root():
    return {
        "service": "FastAPI Dapr Agent",
        "version": "1.0.0",
        "endpoints": {
            "health": "/health",
            "readiness": "/ready",
            "publish": "/publish",
            "subscribe": "/subscribe",
            "state": "/state",
            "db_query": "/db/query"
        }
    }
EOF
```

#### 5.3 Create Requirements File

```bash
cat <<EOF > requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
httpx==0.25.1
pydantic==2.5.0
asyncpg==0.29.0
EOF
```

#### 5.4 Create Dockerfile

```bash
cat <<'EOF' > Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ ./app/

# Expose port
EXPOSE 8000

# Run the application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

### 6. Build and Load Docker Image

```bash
# Build Docker image
docker build -t fastapi-dapr-agent:latest /tmp/fastapi-dapr-agent

# Load image into Minikube
minikube image load fastapi-dapr-agent:latest

# Verify image is loaded
minikube image ls | grep fastapi-dapr-agent
```

### 7. Create Kubernetes Manifests

#### 7.1 Create Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-dapr-agent
  namespace: fastapi-app
  labels:
    app: fastapi-dapr-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastapi-dapr-agent
  template:
    metadata:
      labels:
        app: fastapi-dapr-agent
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "fastapi-dapr-agent"
        dapr.io/app-port: "8000"
        dapr.io/log-level: "info"
        dapr.io/enable-api-logging: "true"
    spec:
      containers:
      - name: fastapi
        image: fastapi-dapr-agent:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: DAPR_HTTP_PORT
          value: "3500"
        - name: DAPR_GRPC_PORT
          value: "50001"
        - name: POSTGRES_HOST
          value: "postgres-postgresql.postgres.svc.cluster.local"
        - name: POSTGRES_PORT
          value: "5432"
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: password
        - name: POSTGRES_DATABASE
          value: "postgres"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
EOF
```

#### 7.2 Create Secret for PostgreSQL Credentials

```bash
# Get PostgreSQL password from existing secret
POSTGRES_PASSWORD=$(kubectl get secret -n postgres postgres-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode)

# Create secret in fastapi-app namespace
kubectl create secret generic postgres-credentials \
  --from-literal=password="${POSTGRES_PASSWORD}" \
  --namespace=fastapi-app \
  --dry-run=client -o yaml | kubectl apply -f -
```

#### 7.3 Create Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: fastapi-dapr-agent
  namespace: fastapi-app
  labels:
    app: fastapi-dapr-agent
spec:
  type: ClusterIP
  selector:
    app: fastapi-dapr-agent
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP
EOF
```

### 8. Wait for Deployment

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/fastapi-dapr-agent -n fastapi-app --timeout=300s

# Check pod status
kubectl get pods -n fastapi-app

# View logs
kubectl logs -n fastapi-app -l app=fastapi-dapr-agent -c fastapi --tail=50

# View Dapr sidecar logs
kubectl logs -n fastapi-app -l app=fastapi-dapr-agent -c daprd --tail=50
```

### 9. Verification & Validation

#### 9.1 Port Forward Service

```bash
# Port forward the FastAPI service
kubectl port-forward -n fastapi-app svc/fastapi-dapr-agent 8000:80 &

# Wait for port forward to establish
sleep 2
```

#### 9.2 Test Health Endpoints

```bash
# Test health endpoint
curl -s http://localhost:8000/health | jq

# Test readiness endpoint
curl -s http://localhost:8000/ready | jq

# Test root endpoint
curl -s http://localhost:8000/ | jq
```

#### 9.3 Test PostgreSQL Connection

```bash
# Test direct PostgreSQL query
curl -s http://localhost:8000/db/query | jq
```

#### 9.4 Test Dapr State Management

```bash
# Save state via Dapr
curl -X POST http://localhost:8000/state \
  -H "Content-Type: application/json" \
  -d '{
    "key": "test-key-1",
    "value": {
      "message": "Hello from FastAPI Dapr Agent",
      "timestamp": "2024-01-01T00:00:00Z"
    }
  }' | jq

# Retrieve state via Dapr
curl -s http://localhost:8000/state/test-key-1 | jq

# Delete state via Dapr
curl -X DELETE http://localhost:8000/state/test-key-1 | jq
```

#### 9.5 Test Kafka Pub/Sub

```bash
# Publish event to Kafka via Dapr
curl -X POST http://localhost:8000/publish \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Test event from FastAPI",
    "metadata": {
      "source": "curl",
      "priority": "high"
    }
  }' | jq

# Check Kafka consumer logs (should show received event)
kubectl logs -n fastapi-app -l app=fastapi-dapr-agent -c fastapi --tail=20 | grep "Received event"
```

#### 9.6 Test Service Invocation via Dapr

```bash
# Invoke service directly via Dapr API (from within cluster)
kubectl exec -it -n fastapi-app deployment/fastapi-dapr-agent -c fastapi -- \
  curl -s http://localhost:3500/v1.0/invoke/fastapi-dapr-agent/method/health | jq
```

#### 9.7 Verify Dapr Components

```bash
# Check Dapr components are loaded
kubectl get components -n fastapi-app

# Describe Kafka pubsub component
kubectl describe component kafka-pubsub -n fastapi-app

# Describe PostgreSQL state store component
kubectl describe component postgres-statestore -n fastapi-app
```

### 10. Complete Validation Script

```bash
# Run comprehensive validation
cat <<'VALIDATION_SCRIPT' | bash
#!/bin/bash
set -e

echo "=== FastAPI Dapr Agent Validation ==="
echo

# Check deployment
echo "1. Checking deployment status..."
kubectl get deployment fastapi-dapr-agent -n fastapi-app
echo "✓ Deployment exists"
echo

# Check pods
echo "2. Checking pod status..."
POD_STATUS=$(kubectl get pods -n fastapi-app -l app=fastapi-dapr-agent -o jsonpath='{.items[0].status.phase}')
if [ "$POD_STATUS" == "Running" ]; then
  echo "✓ Pod is running"
else
  echo "✗ Pod status: $POD_STATUS"
  exit 1
fi
echo

# Check service
echo "3. Checking service..."
kubectl get svc fastapi-dapr-agent -n fastapi-app
echo "✓ Service exists"
echo

# Check Dapr components
echo "4. Checking Dapr components..."
KAFKA_COMPONENT=$(kubectl get component kafka-pubsub -n fastapi-app -o jsonpath='{.metadata.name}' 2>/dev/null || echo "")
POSTGRES_COMPONENT=$(kubectl get component postgres-statestore -n fastapi-app -o jsonpath='{.metadata.name}' 2>/dev/null || echo "")

if [ -n "$KAFKA_COMPONENT" ]; then
  echo "✓ Kafka pubsub component exists"
else
  echo "✗ Kafka pubsub component not found"
  exit 1
fi

if [ -n "$POSTGRES_COMPONENT" ]; then
  echo "✓ PostgreSQL state store component exists"
else
  echo "✗ PostgreSQL state store component not found"
  exit 1
fi
echo

# Port forward (background)
echo "5. Setting up port forward..."
kubectl port-forward -n fastapi-app svc/fastapi-dapr-agent 8000:80 > /dev/null 2>&1 &
PORT_FORWARD_PID=$!
sleep 3
echo "✓ Port forward established (PID: $PORT_FORWARD_PID)"
echo

# Test health
echo "6. Testing health endpoint..."
HEALTH_RESPONSE=$(curl -s http://localhost:8000/health)
if echo "$HEALTH_RESPONSE" | grep -q "healthy"; then
  echo "✓ Health check passed"
  echo "$HEALTH_RESPONSE" | jq
else
  echo "✗ Health check failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test readiness
echo "7. Testing readiness endpoint..."
READY_RESPONSE=$(curl -s http://localhost:8000/ready)
if echo "$READY_RESPONSE" | grep -q "ok"; then
  echo "✓ Readiness check passed"
  echo "$READY_RESPONSE" | jq
else
  echo "✗ Readiness check failed"
  echo "$READY_RESPONSE" | jq
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test database connection
echo "8. Testing PostgreSQL connection..."
DB_RESPONSE=$(curl -s http://localhost:8000/db/query)
if echo "$DB_RESPONSE" | grep -q "postgres_version"; then
  echo "✓ Database connection successful"
  echo "$DB_RESPONSE" | jq
else
  echo "✗ Database connection failed"
  echo "$DB_RESPONSE" | jq
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test state management
echo "9. Testing Dapr state management..."
STATE_SAVE=$(curl -s -X POST http://localhost:8000/state \
  -H "Content-Type: application/json" \
  -d '{"key":"validation-test","value":{"status":"ok"}}')

if echo "$STATE_SAVE" | grep -q "saved"; then
  echo "✓ State save successful"

  STATE_GET=$(curl -s http://localhost:8000/state/validation-test)
  if echo "$STATE_GET" | grep -q "validation-test"; then
    echo "✓ State retrieval successful"
  else
    echo "✗ State retrieval failed"
  fi
else
  echo "✗ State save failed"
fi
echo

# Test pub/sub
echo "10. Testing Kafka pub/sub..."
PUB_RESPONSE=$(curl -s -X POST http://localhost:8000/publish \
  -H "Content-Type: application/json" \
  -d '{"message":"Validation test","metadata":{"test":true}}')

if echo "$PUB_RESPONSE" | grep -q "published"; then
  echo "✓ Event published to Kafka"
  echo "$PUB_RESPONSE" | jq
else
  echo "✗ Event publishing failed"
fi
echo

# Cleanup
echo "Cleaning up port forward..."
kill $PORT_FORWARD_PID 2>/dev/null || true
echo

echo "==================================="
echo "✓ All validations passed!"
echo "==================================="
VALIDATION_SCRIPT
```

## Validation Criteria

- ✓ Dapr system pods running in dapr-system namespace
- ✓ FastAPI application pod running with Dapr sidecar
- ✓ Dapr components (Kafka pubsub, PostgreSQL state store) created
- ✓ Health endpoint returns healthy status
- ✓ Readiness endpoint confirms all dependencies are ready
- ✓ PostgreSQL connection successful via direct query
- ✓ State can be saved, retrieved, and deleted via Dapr
- ✓ Events can be published to Kafka via Dapr
- ✓ Service invocation works through Dapr sidecar

## Cleanup (Optional)

```bash
# Delete application resources
kubectl delete namespace fastapi-app

# Remove Docker image from Minikube
minikube image rm fastapi-dapr-agent:latest

# Remove local files
rm -rf /tmp/fastapi-dapr-agent

# Uninstall Dapr (if needed)
# dapr uninstall -k
```

## Notes for Claude Code and Goose Agents

### Execution Strategy
- **Claude Code**: Execute bash commands sequentially using the Bash tool
- **Goose**: Use toolkit.run_shell() for each command block
- Both agents should validate each step before proceeding
- Capture and parse JSON responses for validation

### Common Pitfalls
- Ensure Kafka and PostgreSQL are deployed before running this playbook
- Dapr components must be created in the same namespace as the application
- Port forwarding runs in background - track PIDs to cleanup properly
- Image must be loaded into Minikube using `minikube image load`
- PostgreSQL password must be base64 decoded when reading from secrets

### Agent-Specific Hints
- **Wait for pods**: Use `kubectl wait --for=condition=Ready` with appropriate timeout
- **Multi-container pods**: Use `-c` flag to specify container when viewing logs
- **Secret handling**: Extract PostgreSQL password and inject into application namespace
- **Validation**: All curl commands should return HTTP 200 and valid JSON
- **Idempotency**: Use `--dry-run=client -o yaml | kubectl apply -f -` for recreatable resources
- **Debugging**: Check both application logs (`-c fastapi`) and Dapr sidecar logs (`-c daprd`)

### Configuration Notes
- Default Dapr HTTP port: 3500
- Default Dapr gRPC port: 50001
- Application runs on port 8000
- Kafka broker: kafka.kafka.svc.cluster.local:9092
- PostgreSQL: postgres-postgresql.postgres.svc.cluster.local:5432
- Resource limits: 512Mi memory, 500m CPU (adjust for production)
- Liveness probe: /health endpoint, 10s initial delay
- Readiness probe: /ready endpoint, 5s initial delay

### Testing Scenarios

#### Scenario 1: State Persistence
1. Save multiple keys with different values
2. Restart the pod
3. Verify state persists after restart

#### Scenario 2: Event-Driven Flow
1. Publish event to Kafka
2. Check application logs for subscription handler execution
3. Verify event data is processed correctly

#### Scenario 3: Service-to-Service Communication
1. Deploy second service (if testing multi-service architecture)
2. Use Dapr service invocation to call fastapi-dapr-agent
3. Verify cross-service communication works

#### Scenario 4: Fault Tolerance
1. Kill Kafka pod temporarily
2. Verify application remains healthy but readiness check fails
3. Restart Kafka and verify service recovers

### Observability
- **Logs**: `kubectl logs -n fastapi-app -l app=fastapi-dapr-agent -c fastapi --tail=100`
- **Dapr logs**: `kubectl logs -n fastapi-app -l app=fastapi-dapr-agent -c daprd --tail=100`
- **Events**: `kubectl get events -n fastapi-app --sort-by='.lastTimestamp'`
- **Metrics**: Access Dapr metrics at http://localhost:9090/metrics (when port-forwarded)

### Production Considerations
- Enable TLS for Dapr components
- Add authentication for PostgreSQL and Kafka
- Increase replica count for high availability
- Configure horizontal pod autoscaling
- Enable Dapr observability (metrics, tracing, logging)
- Use external secrets management (Azure Key Vault, AWS Secrets Manager)
- Configure resource quotas and limits
- Add network policies for security
- Enable persistent volumes for stateful components

## References
- Dapr Documentation: https://docs.dapr.io/
- FastAPI Documentation: https://fastapi.tiangolo.com/
- Dapr Python SDK: https://github.com/dapr/python-sdk
- Kafka Dapr Component: https://docs.dapr.io/reference/components-reference/supported-pubsub/setup-apache-kafka/
- PostgreSQL Dapr Component: https://docs.dapr.io/reference/components-reference/supported-state-stores/setup-postgresql/
- Kubernetes Best Practices: https://kubernetes.io/docs/concepts/configuration/overview/
