---
title: Code Runner Service
description: Deploy FastAPI-based Python code execution service to Kubernetes
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
---

# Code Runner Service

## Goal
Deploy a FastAPI-based code execution service to Kubernetes (Minikube) that safely executes Python code with timeout controls and returns execution results.

This deployment serves as a reference implementation for code execution services in Hackathon III.

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

### 2. Create Code Runner Application

#### 2.1 Create Project Directory

```bash
# Create project directory
mkdir -p /tmp/code-runner-service
cd /tmp/code-runner-service
```

#### 2.2 Create Requirements File

```bash
cat <<'EOF' > requirements.txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
EOF
```

#### 2.3 Create Code Runner Application

```bash
cat <<'EOF' > main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import subprocess
import logging
import tempfile
import os
from typing import Optional

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Code Runner Service", version="1.0.0")

# Configuration
MAX_EXECUTION_TIME = 10  # seconds
MAX_OUTPUT_SIZE = 10000  # characters

# Models
class CodeExecutionRequest(BaseModel):
    code: str
    timeout: Optional[int] = 5  # seconds, default 5

class CodeExecutionResponse(BaseModel):
    stdout: str
    stderr: str
    exit_code: int
    timed_out: bool = False

def execute_python_code(code: str, timeout: int) -> CodeExecutionResponse:
    """
    Execute Python code in a subprocess with timeout.

    Args:
        code: Python code to execute
        timeout: Maximum execution time in seconds

    Returns:
        CodeExecutionResponse with stdout, stderr, exit_code, and timed_out flag
    """
    # Validate timeout
    if timeout > MAX_EXECUTION_TIME:
        timeout = MAX_EXECUTION_TIME
        logger.warning(f"Timeout capped at {MAX_EXECUTION_TIME} seconds")

    if timeout <= 0:
        timeout = 1

    # Create temporary file for code
    try:
        with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as temp_file:
            temp_file.write(code)
            temp_file_path = temp_file.name

        logger.info(f"Executing Python code from {temp_file_path} with timeout {timeout}s")

        # Execute the code
        try:
            result = subprocess.run(
                ['python3', temp_file_path],
                capture_output=True,
                text=True,
                timeout=timeout
            )

            # Truncate output if too large
            stdout = result.stdout[:MAX_OUTPUT_SIZE]
            stderr = result.stderr[:MAX_OUTPUT_SIZE]

            if len(result.stdout) > MAX_OUTPUT_SIZE:
                stdout += "\n... (output truncated)"
            if len(result.stderr) > MAX_OUTPUT_SIZE:
                stderr += "\n... (output truncated)"

            logger.info(f"Code execution completed with exit code {result.returncode}")

            return CodeExecutionResponse(
                stdout=stdout,
                stderr=stderr,
                exit_code=result.returncode,
                timed_out=False
            )

        except subprocess.TimeoutExpired:
            logger.warning(f"Code execution timed out after {timeout}s")
            return CodeExecutionResponse(
                stdout="",
                stderr=f"Execution timed out after {timeout} seconds",
                exit_code=-1,
                timed_out=True
            )

        except Exception as e:
            logger.error(f"Code execution error: {e}")
            return CodeExecutionResponse(
                stdout="",
                stderr=f"Execution error: {str(e)}",
                exit_code=-1,
                timed_out=False
            )

    finally:
        # Cleanup temporary file
        try:
            if 'temp_file_path' in locals():
                os.unlink(temp_file_path)
        except Exception as e:
            logger.error(f"Failed to cleanup temp file: {e}")

# Health endpoint
@app.get("/health")
async def health():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "service": "code-runner-service",
        "version": "1.0.0"
    }

# Readiness endpoint
@app.get("/ready")
async def readiness():
    """Readiness check endpoint"""
    # Test Python execution
    try:
        result = subprocess.run(
            ['python3', '--version'],
            capture_output=True,
            text=True,
            timeout=2
        )
        python_available = result.returncode == 0
    except Exception:
        python_available = False

    return {
        "status": "ready" if python_available else "not_ready",
        "python_available": python_available,
        "max_timeout": MAX_EXECUTION_TIME
    }

# Code execution endpoint
@app.post("/run", response_model=CodeExecutionResponse)
async def run_code(request: CodeExecutionRequest):
    """
    Execute Python code and return results.

    Security Note: This endpoint executes arbitrary code. In production,
    additional sandboxing and security measures should be implemented.
    """
    if not request.code or not request.code.strip():
        raise HTTPException(
            status_code=400,
            detail="Code cannot be empty"
        )

    logger.info(f"Received code execution request (timeout: {request.timeout}s)")
    logger.debug(f"Code: {request.code[:100]}...")  # Log first 100 chars

    try:
        result = execute_python_code(request.code, request.timeout)
        return result
    except Exception as e:
        logger.error(f"Failed to execute code: {e}")
        raise HTTPException(
            status_code=500,
            detail=f"Code execution failed: {str(e)}"
        )

# Root endpoint
@app.get("/")
async def root():
    """Root endpoint with API information"""
    return {
        "service": "Code Runner Service",
        "version": "1.0.0",
        "description": "Execute Python code with timeout controls",
        "endpoints": {
            "health": "/health",
            "ready": "/ready",
            "run": "/run (POST)"
        },
        "limits": {
            "max_timeout": MAX_EXECUTION_TIME,
            "max_output_size": MAX_OUTPUT_SIZE
        }
    }
EOF
```

### 3. Create Dockerfile

```bash
cat <<'EOF' > Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY main.py .

# Create non-root user for security
RUN useradd -m -u 1000 coderunner && \
    chown -R coderunner:coderunner /app

# Switch to non-root user
USER coderunner

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
*.tmp
EOF
```

### 4. Build and Load Docker Image

```bash
# Build Docker image
docker build -t code-runner-service:latest /tmp/code-runner-service

# Load image into Minikube
minikube image load code-runner-service:latest

# Verify image is loaded
minikube image ls | grep code-runner-service
```

### 5. Create Kubernetes Manifests

#### 5.1 Create Namespace

```bash
# Create namespace for code runner
kubectl create namespace code-runner --dry-run=client -o yaml | kubectl apply -f -
```

#### 5.2 Create Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: code-runner-service
  namespace: code-runner
  labels:
    app: code-runner-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: code-runner-service
  template:
    metadata:
      labels:
        app: code-runner-service
    spec:
      containers:
      - name: code-runner
        image: code-runner-service:latest
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
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
EOF
```

#### 5.3 Create Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: code-runner-service
  namespace: code-runner
  labels:
    app: code-runner-service
spec:
  type: ClusterIP
  selector:
    app: code-runner-service
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
kubectl wait --for=condition=available deployment/code-runner-service -n code-runner --timeout=300s

# Check pod status
kubectl get pods -n code-runner

# View logs
kubectl logs -n code-runner -l app=code-runner-service --tail=50
```

### 7. Verification & Validation

#### 7.1 Port Forward Service

```bash
# Port forward the code runner service
kubectl port-forward -n code-runner svc/code-runner-service 8000:80 &

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

# Test code execution - simple print
curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"print(\"Hello, World!\")","timeout":5}'

# Test code execution - calculation
curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"result = 2 + 2\nprint(f\"Result: {result}\")","timeout":5}'

# Test code execution - loop
curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"for i in range(5):\n    print(i)","timeout":5}'

# Test code execution - error handling
curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"x = 1 / 0","timeout":5}'

# Test code execution - timeout (this will timeout)
curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"import time\ntime.sleep(10)\nprint(\"Done\")","timeout":2}'
```

#### 7.3 Complete Validation Script

```bash
cat <<'VALIDATION_SCRIPT' | bash
#!/bin/bash
set -e

echo "=== Code Runner Service Validation ==="
echo

# Check deployment
echo "1. Checking deployment status..."
kubectl get deployment code-runner-service -n code-runner
echo "✓ Deployment exists"
echo

# Check pods
echo "2. Checking pod status..."
POD_STATUS=$(kubectl get pods -n code-runner -l app=code-runner-service -o jsonpath='{.items[0].status.phase}')
if [ "$POD_STATUS" == "Running" ]; then
  echo "✓ Pod is running"
else
  echo "✗ Pod status: $POD_STATUS"
  exit 1
fi
echo

# Check service
echo "3. Checking service..."
kubectl get svc code-runner-service -n code-runner
echo "✓ Service exists"
echo

# Port forward (background)
echo "4. Setting up port forward..."
kubectl port-forward -n code-runner svc/code-runner-service 8000:80 > /dev/null 2>&1 &
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

# Test simple code execution
echo "7. Testing code execution - simple print..."
EXEC_RESPONSE=$(curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"print(\"Test\")","timeout":5}')
if echo "$EXEC_RESPONSE" | grep -q "Test"; then
  echo "✓ Simple execution passed"
  echo "$EXEC_RESPONSE"
else
  echo "✗ Simple execution failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test calculation
echo "8. Testing code execution - calculation..."
CALC_RESPONSE=$(curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"print(10 + 20)","timeout":5}')
if echo "$CALC_RESPONSE" | grep -q "30"; then
  echo "✓ Calculation execution passed"
  echo "$CALC_RESPONSE"
else
  echo "✗ Calculation execution failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test error handling
echo "9. Testing code execution - error handling..."
ERROR_RESPONSE=$(curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"x = 1 / 0","timeout":5}')
if echo "$ERROR_RESPONSE" | grep -q "ZeroDivisionError"; then
  echo "✓ Error handling passed"
  echo "$ERROR_RESPONSE"
else
  echo "✗ Error handling failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test timeout
echo "10. Testing code execution - timeout..."
TIMEOUT_RESPONSE=$(curl -s -X POST http://localhost:8000/run \
  -H "Content-Type: application/json" \
  -d '{"code":"import time\ntime.sleep(10)","timeout":2}')
if echo "$TIMEOUT_RESPONSE" | grep -q "timed_out.*true"; then
  echo "✓ Timeout handling passed"
  echo "$TIMEOUT_RESPONSE"
else
  echo "✗ Timeout handling failed"
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
kubectl port-forward -n code-runner svc/code-runner-service 8000:80

# Method 2: From within cluster
# Service is accessible at: code-runner-service.code-runner.svc.cluster.local:80
```

## Validation Criteria

- ✓ Code runner service builds and starts successfully
- ✓ Docker image created and loaded into Minikube
- ✓ Deployment created with 1 replica running
- ✓ Pod status is Running (1/1 containers ready)
- ✓ Service created with ClusterIP type
- ✓ Health endpoint returns healthy status
- ✓ Readiness endpoint confirms Python is available
- ✓ Code execution endpoint successfully runs Python code
- ✓ stdout and stderr are captured correctly
- ✓ Exit codes are reported accurately
- ✓ Timeout mechanism works correctly
- ✓ Error handling captures exceptions
- ✓ Liveness and readiness probes configured and passing

## Cleanup (Optional)

```bash
# Delete application resources
kubectl delete namespace code-runner

# Remove Docker image from Minikube
minikube image rm code-runner-service:latest

# Remove local files
rm -rf /tmp/code-runner-service
```

## Notes for Claude Code and Goose Agents

### Execution Strategy
- **Claude Code**: Execute bash commands sequentially using the Bash tool
- **Goose**: Use toolkit.run_shell() for each command block
- Both agents should validate each step before proceeding
- Capture and parse JSON responses for validation

### Common Pitfalls
- Code execution requires Python to be available in the container
- Timeout must be reasonable (default max is 10 seconds)
- Large output will be truncated at MAX_OUTPUT_SIZE
- Temporary files must be cleaned up after execution
- Image must be loaded into Minikube using `minikube image load`

### Agent-Specific Hints
- **Wait for deployment**: Use `kubectl wait --for=condition=available` with appropriate timeout
- **Validation**: All curl commands should return HTTP 200 and valid JSON
- **Idempotency**: Use `--dry-run=client -o yaml | kubectl apply -f -` for recreatable resources
- **Debugging**: Check pod logs with `kubectl logs -n code-runner -l app=code-runner-service`
- **Testing code**: Use POST requests with JSON body containing "code" and "timeout" fields

### Configuration Notes
- Application runs on port 8000
- Service exposes port 80, forwards to container port 8000
- Maximum execution timeout: 10 seconds (configurable)
- Maximum output size: 10,000 characters (truncated if exceeded)
- Resource limits: 1Gi memory, 1000m CPU
- Resource requests: 256Mi memory, 200m CPU
- Runs as non-root user (UID 1000) for security
- Liveness probe: /health, 10s initial delay
- Readiness probe: /ready, 5s initial delay

### Security Considerations

**WARNING**: This service executes arbitrary code and should be used with caution.

Current security measures:
1. **Non-root user**: Container runs as UID 1000
2. **Timeout controls**: Maximum 10-second execution time
3. **Output limits**: Truncate large outputs to prevent memory issues
4. **Resource limits**: Kubernetes resource constraints (1Gi memory, 1 CPU)
5. **Temporary file cleanup**: Code files deleted after execution

**NOT included** (would be required for production):
- Network isolation (no outbound connections)
- File system restrictions (read-only filesystem)
- Process isolation (containerization only)
- Input validation (code content filtering)
- Rate limiting
- Authentication/authorization
- Audit logging

### Testing Scenarios

#### Scenario 1: Basic Execution
1. Submit simple print statement
2. Verify stdout contains expected output
3. Check exit_code is 0

#### Scenario 2: Computation
1. Submit arithmetic operations
2. Verify results are correct
3. Test multiple operations

#### Scenario 3: Error Handling
1. Submit code with syntax errors
2. Submit code with runtime errors (division by zero)
3. Verify stderr contains error messages
4. Check exit_code is non-zero

#### Scenario 4: Timeout
1. Submit code with long sleep
2. Set timeout lower than sleep duration
3. Verify timed_out flag is true
4. Check exit_code is -1

#### Scenario 5: Output Limits
1. Submit code generating large output
2. Verify output is truncated
3. Check truncation message appears

### Response Format

```json
{
  "stdout": "output from print statements",
  "stderr": "error messages if any",
  "exit_code": 0,
  "timed_out": false
}
```

- **stdout**: Standard output from executed code
- **stderr**: Standard error from executed code
- **exit_code**: Process exit code (0 = success, -1 = timeout/error, other = Python exit code)
- **timed_out**: Boolean flag indicating if execution exceeded timeout

### Observability
- **Logs**: `kubectl logs -n code-runner -l app=code-runner-service --tail=100 -f`
- **Events**: `kubectl get events -n code-runner --sort-by='.lastTimestamp'`
- **Pod status**: `kubectl describe pod -n code-runner -l app=code-runner-service`
- **Resource usage**: `kubectl top pod -n code-runner`

### Extension Points
To enhance security:
1. Add network policies to block outbound connections
2. Implement read-only root filesystem
3. Add input validation and sanitization
4. Implement rate limiting per client
5. Add authentication/authorization
6. Use more restrictive securityContext

To add features:
1. Support multiple programming languages
2. Add package installation support
3. Implement code caching
4. Add execution history/logging
5. Support file upload/download

## References
- FastAPI Documentation: https://fastapi.tiangolo.com/
- Python subprocess: https://docs.python.org/3/library/subprocess.html
- Kubernetes Security Context: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
- Container Security Best Practices: https://kubernetes.io/docs/concepts/security/pod-security-standards/
