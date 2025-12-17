---
title: MCP Python Server
description: Deploy FastAPI-based Model Context Protocol (MCP) server to Kubernetes
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
---

# MCP Python Server

## Goal
Deploy a FastAPI-based Model Context Protocol (MCP) server to Kubernetes (Minikube) with health checks and example MCP tool execution endpoint.

This deployment serves as a reference implementation for MCP servers in Hackathon III.

## Prerequisites Check
Before proceeding, verify:
1. Minikube is running
2. kubectl is configured and accessible
3. Docker is installed for building images
4. Python 3.11+ is installed (for local verification)

## Instructions

### 1. Verify Prerequisites

```bash
# Check Minikube status
minikube status

# Verify kubectl access
kubectl cluster-info
kubectl get nodes

# Check Docker version
docker --version

# Check Python version
python3 --version
```

### 2. Create MCP Server Application

#### 2.1 Create Project Directory

```bash
# Create project directory
mkdir -p /tmp/mcp-python-server
cd /tmp/mcp-python-server
```

#### 2.2 Create Requirements File

```bash
cat <<'EOF' > requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
EOF
```

#### 2.3 Create MCP Server Application

```bash
cat <<'EOF' > main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict, Any, Optional
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="MCP Python Server", version="1.0.0")

# Models
class MCPTool(BaseModel):
    name: str
    description: str
    parameters: Dict[str, Any]

class MCPResource(BaseModel):
    uri: str
    name: str
    mimeType: str

class MCPExecuteRequest(BaseModel):
    tool: str
    arguments: Dict[str, Any] = {}

class MCPExecuteResponse(BaseModel):
    result: Any
    metadata: Optional[Dict[str, Any]] = {}

# Example tools registry
TOOLS: List[MCPTool] = [
    MCPTool(
        name="echo",
        description="Echo back the provided message",
        parameters={
            "type": "object",
            "properties": {
                "message": {
                    "type": "string",
                    "description": "Message to echo"
                }
            },
            "required": ["message"]
        }
    ),
    MCPTool(
        name="calculate",
        description="Perform basic arithmetic calculation",
        parameters={
            "type": "object",
            "properties": {
                "operation": {
                    "type": "string",
                    "enum": ["add", "subtract", "multiply", "divide"],
                    "description": "Arithmetic operation"
                },
                "a": {
                    "type": "number",
                    "description": "First operand"
                },
                "b": {
                    "type": "number",
                    "description": "Second operand"
                }
            },
            "required": ["operation", "a", "b"]
        }
    )
]

# Example resources registry
RESOURCES: List[MCPResource] = [
    MCPResource(
        uri="mcp://demo/status",
        name="Server Status",
        mimeType="application/json"
    )
]

# Tool execution functions
def execute_echo(arguments: Dict[str, Any]) -> str:
    """Echo tool implementation"""
    message = arguments.get("message", "")
    logger.info(f"Echo tool called with message: {message}")
    return f"Echo: {message}"

def execute_calculate(arguments: Dict[str, Any]) -> float:
    """Calculate tool implementation"""
    operation = arguments.get("operation")
    a = arguments.get("a", 0)
    b = arguments.get("b", 0)

    logger.info(f"Calculate tool called: {operation}({a}, {b})")

    if operation == "add":
        return a + b
    elif operation == "subtract":
        return a - b
    elif operation == "multiply":
        return a * b
    elif operation == "divide":
        if b == 0:
            raise ValueError("Division by zero")
        return a / b
    else:
        raise ValueError(f"Unknown operation: {operation}")

# Tool executor mapping
TOOL_EXECUTORS = {
    "echo": execute_echo,
    "calculate": execute_calculate
}

# Health endpoint
@app.get("/health")
async def health():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "service": "mcp-python-server",
        "version": "1.0.0"
    }

# Readiness endpoint
@app.get("/ready")
async def readiness():
    """Readiness check endpoint"""
    return {
        "status": "ready",
        "tools_count": len(TOOLS),
        "resources_count": len(RESOURCES)
    }

# MCP endpoints
@app.get("/mcp/tools")
async def list_tools():
    """List available MCP tools"""
    return {
        "tools": [tool.dict() for tool in TOOLS]
    }

@app.get("/mcp/resources")
async def list_resources():
    """List available MCP resources"""
    return {
        "resources": [resource.dict() for resource in RESOURCES]
    }

@app.post("/mcp/execute", response_model=MCPExecuteResponse)
async def execute_tool(request: MCPExecuteRequest):
    """Execute an MCP tool"""
    tool_name = request.tool
    arguments = request.arguments

    logger.info(f"Executing tool: {tool_name} with arguments: {arguments}")

    # Check if tool exists
    if tool_name not in TOOL_EXECUTORS:
        raise HTTPException(
            status_code=404,
            detail=f"Tool '{tool_name}' not found"
        )

    try:
        # Execute the tool
        executor = TOOL_EXECUTORS[tool_name]
        result = executor(arguments)

        return MCPExecuteResponse(
            result=result,
            metadata={
                "tool": tool_name,
                "success": True
            }
        )
    except Exception as e:
        logger.error(f"Tool execution failed: {e}")
        raise HTTPException(
            status_code=500,
            detail=f"Tool execution failed: {str(e)}"
        )

@app.get("/mcp/resource/{uri:path}")
async def get_resource(uri: str):
    """Get an MCP resource"""
    logger.info(f"Fetching resource: {uri}")

    # Example resource response
    if uri == "mcp://demo/status":
        return {
            "uri": uri,
            "content": {
                "server": "mcp-python-server",
                "status": "operational",
                "uptime": "available"
            },
            "mimeType": "application/json"
        }

    raise HTTPException(
        status_code=404,
        detail=f"Resource '{uri}' not found"
    )

# Root endpoint
@app.get("/")
async def root():
    """Root endpoint with API information"""
    return {
        "service": "MCP Python Server",
        "version": "1.0.0",
        "protocol": "Model Context Protocol",
        "endpoints": {
            "health": "/health",
            "ready": "/ready",
            "tools": "/mcp/tools",
            "resources": "/mcp/resources",
            "execute": "/mcp/execute",
            "resource": "/mcp/resource/{uri}"
        }
    }
EOF
```

### 3. Create Dockerfile

```bash
cat <<'EOF' > Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY main.py .

# Expose port
EXPOSE 8000

# Run the application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
EOF
```

#### 3.1 Create .dockerignore

```bash
cat <<'EOF' > .dockerignore
__pycache__
*.pyc
*.pyo
*.pyd
.Python
.git
.gitignore
README.md
.env
EOF
```

### 4. Build and Load Docker Image

```bash
# Build Docker image
docker build -t mcp-python-server:latest /tmp/mcp-python-server

# Load image into Minikube
minikube image load mcp-python-server:latest

# Verify image is loaded
minikube image ls | grep mcp-python-server
```

### 5. Create Kubernetes Manifests

#### 5.1 Create Namespace

```bash
# Create namespace for MCP server
kubectl create namespace mcp-server --dry-run=client -o yaml | kubectl apply -f -
```

#### 5.2 Create Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-python-server
  namespace: mcp-server
  labels:
    app: mcp-python-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-python-server
  template:
    metadata:
      labels:
        app: mcp-python-server
    spec:
      containers:
      - name: mcp-server
        image: mcp-python-server:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: LOG_LEVEL
          value: "info"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
EOF
```

#### 5.3 Create Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mcp-python-server
  namespace: mcp-server
  labels:
    app: mcp-python-server
spec:
  type: ClusterIP
  selector:
    app: mcp-python-server
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP
EOF
```

### 6. Wait for Deployment

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/mcp-python-server -n mcp-server --timeout=300s

# Check pod status
kubectl get pods -n mcp-server

# View logs
kubectl logs -n mcp-server -l app=mcp-python-server --tail=50
```

### 7. Verification & Validation

#### 7.1 Port Forward Service

```bash
# Port forward the MCP server
kubectl port-forward -n mcp-server svc/mcp-python-server 8000:80 &

# Wait for port forward to establish
sleep 3
```

#### 7.2 Test Endpoints

```bash
# Test health endpoint
curl -s http://localhost:8000/health

# Test readiness endpoint
curl -s http://localhost:8000/ready

# Test root endpoint
curl -s http://localhost:8000/

# List available tools
curl -s http://localhost:8000/mcp/tools

# List available resources
curl -s http://localhost:8000/mcp/resources

# Execute echo tool
curl -s -X POST http://localhost:8000/mcp/execute \
  -H "Content-Type: application/json" \
  -d '{"tool":"echo","arguments":{"message":"Hello MCP"}}'

# Execute calculate tool (addition)
curl -s -X POST http://localhost:8000/mcp/execute \
  -H "Content-Type: application/json" \
  -d '{"tool":"calculate","arguments":{"operation":"add","a":10,"b":5}}'

# Execute calculate tool (multiplication)
curl -s -X POST http://localhost:8000/mcp/execute \
  -H "Content-Type: application/json" \
  -d '{"tool":"calculate","arguments":{"operation":"multiply","a":7,"b":8}}'

# Get resource
curl -s http://localhost:8000/mcp/resource/mcp://demo/status
```

#### 7.3 Complete Validation Script

```bash
cat <<'VALIDATION_SCRIPT' | bash
#!/bin/bash
set -e

echo "=== MCP Python Server Validation ==="
echo

# Check deployment
echo "1. Checking deployment status..."
kubectl get deployment mcp-python-server -n mcp-server
echo "✓ Deployment exists"
echo

# Check pods
echo "2. Checking pod status..."
POD_STATUS=$(kubectl get pods -n mcp-server -l app=mcp-python-server -o jsonpath='{.items[0].status.phase}')
if [ "$POD_STATUS" == "Running" ]; then
  echo "✓ Pod is running"
else
  echo "✗ Pod status: $POD_STATUS"
  exit 1
fi
echo

# Check service
echo "3. Checking service..."
kubectl get svc mcp-python-server -n mcp-server
echo "✓ Service exists"
echo

# Port forward (background)
echo "4. Setting up port forward..."
kubectl port-forward -n mcp-server svc/mcp-python-server 8000:80 > /dev/null 2>&1 &
PORT_FORWARD_PID=$!
sleep 3
echo "✓ Port forward established (PID: $PORT_FORWARD_PID)"
echo

# Test health
echo "5. Testing health endpoint..."
HEALTH_RESPONSE=$(curl -s http://localhost:8000/health)
if echo "$HEALTH_RESPONSE" | grep -q "healthy"; then
  echo "✓ Health check passed"
  echo "$HEALTH_RESPONSE"
else
  echo "✗ Health check failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test readiness
echo "6. Testing readiness endpoint..."
READY_RESPONSE=$(curl -s http://localhost:8000/ready)
if echo "$READY_RESPONSE" | grep -q "ready"; then
  echo "✓ Readiness check passed"
  echo "$READY_RESPONSE"
else
  echo "✗ Readiness check failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test MCP tools list
echo "7. Testing MCP tools endpoint..."
TOOLS_RESPONSE=$(curl -s http://localhost:8000/mcp/tools)
if echo "$TOOLS_RESPONSE" | grep -q "tools"; then
  echo "✓ MCP tools listed"
  echo "$TOOLS_RESPONSE"
else
  echo "✗ MCP tools failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test MCP execute (echo tool)
echo "8. Testing MCP execute - echo tool..."
EXECUTE_RESPONSE=$(curl -s -X POST http://localhost:8000/mcp/execute \
  -H "Content-Type: application/json" \
  -d '{"tool":"echo","arguments":{"message":"Test Message"}}')
if echo "$EXECUTE_RESPONSE" | grep -q "Echo: Test Message"; then
  echo "✓ Echo tool executed successfully"
  echo "$EXECUTE_RESPONSE"
else
  echo "✗ Echo tool execution failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test MCP execute (calculate tool)
echo "9. Testing MCP execute - calculate tool..."
CALC_RESPONSE=$(curl -s -X POST http://localhost:8000/mcp/execute \
  -H "Content-Type: application/json" \
  -d '{"tool":"calculate","arguments":{"operation":"add","a":15,"b":25}}')
if echo "$CALC_RESPONSE" | grep -q "40"; then
  echo "✓ Calculate tool executed successfully"
  echo "$CALC_RESPONSE"
else
  echo "✗ Calculate tool execution failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test MCP resource
echo "10. Testing MCP resource endpoint..."
RESOURCE_RESPONSE=$(curl -s http://localhost:8000/mcp/resource/mcp://demo/status)
if echo "$RESOURCE_RESPONSE" | grep -q "operational"; then
  echo "✓ MCP resource retrieved"
  echo "$RESOURCE_RESPONSE"
else
  echo "✗ MCP resource failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
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

### 8. Access Application

```bash
# Method 1: Port forward (for local access)
kubectl port-forward -n mcp-server svc/mcp-python-server 8000:80

# Method 2: From within cluster
# Service is accessible at: mcp-python-server.mcp-server.svc.cluster.local:80
```

## Validation Criteria

- ✓ MCP server builds and starts successfully
- ✓ Docker image created and loaded into Minikube
- ✓ Deployment created with 1 replica running
- ✓ Pod status is Running (1/1 containers ready)
- ✓ Service created with ClusterIP type
- ✓ Health endpoint returns healthy status
- ✓ Readiness endpoint returns ready status with tool counts
- ✓ MCP tools endpoint lists available tools
- ✓ MCP resources endpoint lists available resources
- ✓ MCP execute endpoint successfully executes tools
- ✓ MCP resource endpoint retrieves resources
- ✓ Liveness and readiness probes configured and passing

## Cleanup (Optional)

```bash
# Delete application resources
kubectl delete namespace mcp-server

# Remove Docker image from Minikube
minikube image rm mcp-python-server:latest

# Remove local files
rm -rf /tmp/mcp-python-server
```

## Notes for Claude Code and Goose Agents

### Execution Strategy
- **Claude Code**: Execute bash commands sequentially using the Bash tool
- **Goose**: Use toolkit.run_shell() for each command block
- Both agents should validate each step before proceeding
- Capture and parse JSON responses for validation

### Common Pitfalls
- FastAPI requires uvicorn to run
- MCP execute endpoint expects JSON body with tool and arguments
- Port 8000 is the default for this server
- Tool executors must be registered in TOOL_EXECUTORS dict
- Image must be loaded into Minikube using `minikube image load`

### Agent-Specific Hints
- **Wait for deployment**: Use `kubectl wait --for=condition=available` with appropriate timeout
- **Validation**: All curl commands should return HTTP 200 and valid JSON
- **Idempotency**: Use `--dry-run=client -o yaml | kubectl apply -f -` for recreatable resources
- **Debugging**: Check pod logs with `kubectl logs -n mcp-server -l app=mcp-python-server`
- **Testing tools**: Use POST requests with JSON body for /mcp/execute

### Configuration Notes
- Application runs on port 8000
- Service exposes port 80, forwards to container port 8000
- Two example tools included: echo and calculate
- One example resource included: mcp://demo/status
- Resource limits: 512Mi memory, 500m CPU
- Liveness probe: /health, 10s initial delay
- Readiness probe: /ready, 5s initial delay

### MCP Protocol Implementation

#### Tools
Tools are executable functions that the MCP server provides. Each tool has:
- **name**: Unique identifier
- **description**: What the tool does
- **parameters**: JSON schema defining expected arguments

Example tools included:
1. **echo**: Returns the provided message
2. **calculate**: Performs basic arithmetic (add, subtract, multiply, divide)

#### Resources
Resources are data sources that the MCP server can provide. Each resource has:
- **uri**: Unique resource identifier
- **name**: Human-readable name
- **mimeType**: Content type

Example resource included:
- **mcp://demo/status**: Server status information

#### Execution Flow
1. Client calls `/mcp/tools` to discover available tools
2. Client calls `/mcp/execute` with tool name and arguments
3. Server validates tool exists and executes it
4. Server returns result with metadata

### Testing Scenarios

#### Scenario 1: Tool Discovery
1. Query `/mcp/tools` endpoint
2. Verify all registered tools are returned
3. Check tool schemas are valid

#### Scenario 2: Tool Execution
1. Execute echo tool with message
2. Execute calculate tool with different operations
3. Verify results match expected values
4. Test error handling (division by zero, invalid tool)

#### Scenario 3: Resource Access
1. Query `/mcp/resources` to list available resources
2. Fetch specific resource by URI
3. Verify resource content is returned

#### Scenario 4: Health Monitoring
1. Check health endpoint remains healthy
2. Verify readiness reports correct tool/resource counts
3. Confirm Kubernetes probes are passing

### Observability
- **Logs**: `kubectl logs -n mcp-server -l app=mcp-python-server --tail=100 -f`
- **Events**: `kubectl get events -n mcp-server --sort-by='.lastTimestamp'`
- **Pod status**: `kubectl describe pod -n mcp-server -l app=mcp-python-server`
- **Resource usage**: `kubectl top pod -n mcp-server`

### Extension Points
To add new tools:
1. Define tool schema in TOOLS list
2. Implement executor function
3. Register executor in TOOL_EXECUTORS dict
4. Restart deployment

To add new resources:
1. Define resource in RESOURCES list
2. Add resource handler in get_resource endpoint
3. Restart deployment

## References
- Model Context Protocol: https://modelcontextprotocol.io/
- FastAPI Documentation: https://fastapi.tiangolo.com/
- Kubernetes Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Python Type Hints: https://docs.python.org/3/library/typing.html
