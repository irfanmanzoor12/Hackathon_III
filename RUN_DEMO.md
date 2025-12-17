# RUN_DEMO.md

**Hackathon III: 10-Minute System Validation**

This guide enables judges to deploy and validate the entire CAPS playbook system from scratch.

---

## Prerequisites (5 minutes to install if missing)

```bash
# Required
docker --version          # 20.10+
minikube version         # 1.30+
kubectl version --client # 1.27+
node --version           # 18+

# Install Minikube (Ubuntu/Debian)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

---

## 1. Cluster Startup (one-time)

```bash
# Start Minikube with adequate resources
minikube start --cpus=4 --memory=8192 --disk-size=20g

# Verify cluster is ready
kubectl cluster-info
kubectl get nodes
```

**Expected**: Minikube running, kubectl connected.

---

## 2. Deploy Services (exact order)

### 2.1 PostgreSQL (state layer)

```bash
# Execute playbook
cd playbooks/postgres-k8s-setup
# Follow playbook.md steps 1-6

# Quick validation
kubectl get pods -n postgres
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n postgres --timeout=300s
```

**Expected**: `postgres-postgresql-0` pod Running.

### 2.2 Install Dapr

```bash
# Install Dapr CLI
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
export PATH="$HOME/.local/bin:$PATH"

# Initialize Dapr on Kubernetes
dapr init -k

# Verify
kubectl get pods -n dapr-system
```

**Expected**: 8 Dapr pods Running in `dapr-system` namespace.

### 2.3 FastAPI + Dapr Agent

```bash
# Execute playbook
cd playbooks/fastapi-dapr-agent
# Follow playbook.md steps 1-9 (skip Kafka component if not deployed)

# Quick validation
kubectl get pods -n fastapi-app
kubectl wait --for=condition=available deployment/fastapi-dapr-agent -n fastapi-app --timeout=300s
```

**Expected**: `fastapi-dapr-agent-*` pod with 2/2 containers ready (fastapi + daprd).

### 2.4 Next.js Frontend

```bash
# Execute playbook
cd playbooks/nextjs-k8s-deploy
# Follow playbook.md steps 1-7

# Quick validation
kubectl get pods -n nextjs-app
kubectl wait --for=condition=available deployment/nextjs-k8s-app -n nextjs-app --timeout=300s
```

**Expected**: `nextjs-k8s-app-*` pod Running.

### 2.5 Docusaurus Documentation (optional)

```bash
cd playbooks/docusaurus-deploy
# Follow playbook.md steps 1-7

kubectl get pods -n docusaurus
```

**Expected**: 2 `docusaurus-site-*` pods Running.

---

## 3. Port-Forward Services

Open **4 terminals** and run one command in each:

```bash
# Terminal 1: FastAPI backend
kubectl port-forward -n fastapi-app svc/fastapi-dapr-agent 8000:80

# Terminal 2: Next.js frontend
kubectl port-forward -n nextjs-app svc/nextjs-k8s-app 3000:80

# Terminal 3: PostgreSQL (for direct queries)
kubectl port-forward -n postgres svc/postgres-postgresql 5432:5432

# Terminal 4: Docusaurus (if deployed)
kubectl port-forward -n docusaurus svc/docusaurus-site 8080:80
```

**Note**: Leave terminals running during validation.

---

## 4. Validation Checklist

### 4.1 FastAPI Backend Health

```bash
# Health check
curl http://localhost:8000/health
# Expected: {"status":"healthy","service":"fastapi-dapr-agent"}

# Readiness check
curl http://localhost:8000/ready
# Expected: {"api":"ok","database":"ok","dapr":"ok"}

# Database connectivity
curl http://localhost:8000/db/query
# Expected: {"status":"ok","postgres_version":"PostgreSQL 18.1...","table_count":2}

# Dapr state management (save)
curl -X POST http://localhost:8000/state \
  -H "Content-Type: application/json" \
  -d '{"key":"demo-key","value":{"message":"Hackathon III demo"}}'
# Expected: {"status":"saved","key":"demo-key"}

# Dapr state management (retrieve)
curl http://localhost:8000/state/demo-key
# Expected: {"key":"demo-key","value":{"message":"Hackathon III demo"}}
```

### 4.2 Next.js Frontend

**Browser**: Open http://localhost:3000

**Expected**:
- Home page displays: "Next.js 14 on Kubernetes"
- Navigation works (App Router)
- Links to health/ready endpoints functional

**CLI Validation**:
```bash
curl http://localhost:3000/api/health
# Expected: {"status":"healthy","service":"nextjs-k8s-app","timestamp":"..."}

curl http://localhost:3000/api/ready
# Expected: {"server":"ok","timestamp":"..."}
```

### 4.3 Docusaurus Documentation (if deployed)

**Browser**: Open http://localhost:8080

**Expected**:
- Documentation site loads
- "Docusaurus on Kubernetes" homepage
- Docs and blog sections accessible

**CLI Validation**:
```bash
curl http://localhost:8080/health
# Expected: {"status":"healthy","service":"docusaurus-site"}
```

### 4.4 Kubernetes Infrastructure

```bash
# Check all namespaces
kubectl get namespaces | grep -E "fastapi-app|nextjs-app|postgres|dapr-system|docusaurus"

# Check all pods
kubectl get pods --all-namespaces | grep -E "fastapi|nextjs|postgres|dapr|docusaurus"

# Check Dapr components
kubectl get components -n fastapi-app
# Expected: postgres-statestore component
```

---

## 5. Success Criteria

**You have successfully validated the system if**:

✅ **Backend**: FastAPI health check returns `healthy`
✅ **Database**: PostgreSQL connection confirmed via `/db/query`
✅ **State**: Dapr state save/retrieve works
✅ **Frontend**: Next.js site loads in browser
✅ **Probes**: All Kubernetes readiness/liveness probes passing
✅ **Dapr**: Sidecar injected and operational (2/2 containers)
✅ **Documentation**: Docusaurus site accessible (if deployed)

**Visual Confirmation**:
- Browser shows Next.js homepage
- curl commands return expected JSON
- No pods in CrashLoopBackOff or Error state

---

## 6. Architecture Overview (for reviewers)

**What you just deployed**:

A **cloud-native microservices platform** on Kubernetes featuring:

1. **FastAPI + Dapr microservice** with:
   - Dapr sidecar for service mesh capabilities
   - PostgreSQL state management via Dapr components
   - Health/readiness endpoints for observability
   - Event-driven architecture ready (Kafka integration pending)

2. **Next.js 14 frontend** with:
   - App Router for modern React patterns
   - Standalone build for minimal container size
   - API routes for health checks
   - Production-optimized (multi-stage Docker build)

3. **PostgreSQL database** (Helm-managed):
   - Persistent state layer
   - Dapr state store backend
   - Direct query validation endpoint

4. **Dapr control plane** (8 components):
   - Service invocation, pub/sub, state management
   - Sidecar injection for zero-code integration
   - Cloud-agnostic abstractions

5. **Docusaurus documentation** (optional):
   - Static site with Nginx
   - 2 replicas for high availability
   - Gzip compression and caching

**Key Differentiator**: All services deployed via **CAPS playbooks**—vendor-neutral, executable documentation that enables single-prompt deployments. Not scripts, not manual configs—autonomous, validated, production-ready patterns.

**Why it matters**: This is not a CRUD app. It's a **playbook library** that teaches AI agents how to operate cloud infrastructure. The services are proof of concept; the playbooks are the product.

---

## 7. Troubleshooting

**If pod is not ready**:
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --tail=50
```

**If port-forward fails**:
- Check pod is Running: `kubectl get pods -n <namespace>`
- Kill existing port-forward: `pkill -f "port-forward"`
- Retry command

**If Dapr sidecar missing**:
```bash
# Check namespace has Dapr injection enabled
kubectl get namespace fastapi-app -o jsonpath='{.metadata.labels}'
# Should include: dapr.io/enabled=true
```

**If database connection fails**:
```bash
# Verify PostgreSQL secret exists
kubectl get secret postgres-credentials -n fastapi-app
# If missing, recreate from playbook step 7.2
```

---

## 8. Cleanup

```bash
# Delete all deployed services
kubectl delete namespace fastapi-app nextjs-app postgres docusaurus dapr-system

# Stop Minikube (optional)
minikube stop

# Delete cluster (full reset)
minikube delete
```

---

## Quick Reference

**Service URLs** (with port-forward active):
- FastAPI Backend: http://localhost:8000
- Next.js Frontend: http://localhost:3000
- Docusaurus Docs: http://localhost:8080
- PostgreSQL: localhost:5432

**Health Endpoints**:
- FastAPI: http://localhost:8000/health
- Next.js: http://localhost:3000/api/health
- Docusaurus: http://localhost:8080/health

**Playbook Locations**:
```
playbooks/
├── fastapi-dapr-agent/playbook.md
├── nextjs-k8s-deploy/playbook.md
├── postgres-k8s-setup/playbook.md
├── docusaurus-deploy/playbook.md
├── mcp-python-server/playbook.md
└── code-runner-service/playbook.md
```

---

**Total Time**: 10 minutes (assuming prerequisites installed)
**Validation Time**: 2 minutes (run all curl commands)
**Judge Verdict Time**: 30 seconds (open browser, see it work)

**Hackathon III: Validated. Production-ready. Autonomous.**
