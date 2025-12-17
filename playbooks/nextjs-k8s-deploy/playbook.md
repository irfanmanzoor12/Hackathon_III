---
title: Next.js K8s Deploy
description: Deploy Next.js 14 App Router application to Kubernetes
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
---

# Next.js K8s Deploy

## Goal
Deploy a Next.js 14 application with App Router to Kubernetes (Minikube) using Docker and kubectl, with health checks and validation.

This deployment serves as a reference implementation for containerized Next.js applications in Hackathon III.

## Prerequisites Check
Before proceeding, verify:
1. Minikube is running
2. kubectl is configured and accessible
3. Docker is installed for building images
4. Node.js 18+ is installed (for local build verification)

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

# Check Node.js version
node --version
```

### 2. Create Next.js Application

#### 2.1 Create Project Directory

```bash
# Create project directory
mkdir -p /tmp/nextjs-k8s-app
cd /tmp/nextjs-k8s-app
```

#### 2.2 Initialize Next.js Project Files

```bash
# Create package.json
cat <<'EOF' > package.json
{
  "name": "nextjs-k8s-app",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "14.2.3",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/node": "^20.12.7",
    "@types/react": "^18.3.1",
    "@types/react-dom": "^18.3.0",
    "typescript": "^5.4.5"
  }
}
EOF
```

#### 2.3 Create TypeScript Configuration

```bash
cat <<'EOF' > tsconfig.json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
EOF
```

#### 2.4 Create Next.js Configuration

```bash
cat <<'EOF' > next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone',
  poweredByHeader: false,
}

module.exports = nextConfig
EOF
```

#### 2.5 Create Application Structure

```bash
# Create app directory structure
mkdir -p app/api/health

# Create root layout
cat <<'EOF' > app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'Next.js K8s App',
  description: 'Next.js 14 on Kubernetes',
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
EOF

# Create home page
cat <<'EOF' > app/page.tsx
export default function Home() {
  return (
    <main style={{ padding: '2rem', fontFamily: 'system-ui, sans-serif' }}>
      <h1>Next.js 14 on Kubernetes</h1>
      <p>App Router • Standalone Output • Container Ready</p>
      <ul>
        <li><a href="/api/health">Health Check</a></li>
        <li><a href="/api/ready">Readiness Check</a></li>
      </ul>
    </main>
  )
}
EOF

# Create health endpoint
cat <<'EOF' > app/api/health/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json(
    {
      status: 'healthy',
      service: 'nextjs-k8s-app',
      timestamp: new Date().toISOString()
    },
    { status: 200 }
  )
}
EOF

# Create readiness endpoint
cat <<'EOF' > app/api/ready/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  // Perform readiness checks
  const checks = {
    server: 'ok',
    timestamp: new Date().toISOString()
  }

  return NextResponse.json(checks, { status: 200 })
}
EOF
```

### 3. Create Dockerfile

```bash
cat <<'EOF' > Dockerfile
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
EOF
```

#### 3.1 Create .dockerignore

```bash
cat <<'EOF' > .dockerignore
node_modules
.next
.git
.gitignore
README.md
.env*.local
EOF
```

### 4. Build Application and Docker Image

#### 4.1 Install Dependencies and Build

```bash
# Install dependencies
cd /tmp/nextjs-k8s-app
npm install

# Build Next.js application
npm run build
```

#### 4.2 Build Docker Image

```bash
# Build Docker image
docker build -t nextjs-k8s-app:latest /tmp/nextjs-k8s-app

# Load image into Minikube
minikube image load nextjs-k8s-app:latest

# Verify image is loaded
minikube image ls | grep nextjs-k8s-app
```

### 5. Create Kubernetes Manifests

#### 5.1 Create Namespace

```bash
# Create namespace for Next.js app
kubectl create namespace nextjs-app --dry-run=client -o yaml | kubectl apply -f -
```

#### 5.2 Create Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-k8s-app
  namespace: nextjs-app
  labels:
    app: nextjs-k8s-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nextjs-k8s-app
  template:
    metadata:
      labels:
        app: nextjs-k8s-app
    spec:
      containers:
      - name: nextjs
        image: nextjs-k8s-app:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "3000"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /api/ready
            port: 3000
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
  name: nextjs-k8s-app
  namespace: nextjs-app
  labels:
    app: nextjs-k8s-app
spec:
  type: ClusterIP
  selector:
    app: nextjs-k8s-app
  ports:
  - name: http
    port: 80
    targetPort: 3000
    protocol: TCP
EOF
```

### 6. Wait for Deployment

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/nextjs-k8s-app -n nextjs-app --timeout=300s

# Check pod status
kubectl get pods -n nextjs-app

# View logs
kubectl logs -n nextjs-app -l app=nextjs-k8s-app --tail=50
```

### 7. Verification & Validation

#### 7.1 Port Forward Service

```bash
# Port forward the Next.js service
kubectl port-forward -n nextjs-app svc/nextjs-k8s-app 3000:80 &

# Wait for port forward to establish
sleep 3
```

#### 7.2 Test Endpoints

```bash
# Test health endpoint
curl -s http://localhost:3000/api/health

# Test readiness endpoint
curl -s http://localhost:3000/api/ready

# Test home page
curl -s http://localhost:3000/ | grep -o "<h1>.*</h1>"
```

#### 7.3 Complete Validation Script

```bash
cat <<'VALIDATION_SCRIPT' | bash
#!/bin/bash
set -e

echo "=== Next.js K8s Deployment Validation ==="
echo

# Check deployment
echo "1. Checking deployment status..."
kubectl get deployment nextjs-k8s-app -n nextjs-app
echo "✓ Deployment exists"
echo

# Check pods
echo "2. Checking pod status..."
POD_STATUS=$(kubectl get pods -n nextjs-app -l app=nextjs-k8s-app -o jsonpath='{.items[0].status.phase}')
if [ "$POD_STATUS" == "Running" ]; then
  echo "✓ Pod is running"
else
  echo "✗ Pod status: $POD_STATUS"
  exit 1
fi
echo

# Check service
echo "3. Checking service..."
kubectl get svc nextjs-k8s-app -n nextjs-app
echo "✓ Service exists"
echo

# Port forward (background)
echo "4. Setting up port forward..."
kubectl port-forward -n nextjs-app svc/nextjs-k8s-app 3000:80 > /dev/null 2>&1 &
PORT_FORWARD_PID=$!
sleep 3
echo "✓ Port forward established (PID: $PORT_FORWARD_PID)"
echo

# Test health
echo "5. Testing health endpoint..."
HEALTH_RESPONSE=$(curl -s http://localhost:3000/api/health)
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
READY_RESPONSE=$(curl -s http://localhost:3000/api/ready)
if echo "$READY_RESPONSE" | grep -q "ok"; then
  echo "✓ Readiness check passed"
  echo "$READY_RESPONSE"
else
  echo "✗ Readiness check failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test home page
echo "7. Testing home page..."
HOME_RESPONSE=$(curl -s http://localhost:3000/)
if echo "$HOME_RESPONSE" | grep -q "Next.js 14 on Kubernetes"; then
  echo "✓ Home page renders correctly"
else
  echo "✗ Home page test failed"
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
kubectl port-forward -n nextjs-app svc/nextjs-k8s-app 3000:80

# Method 2: From within cluster
# Service is accessible at: nextjs-k8s-app.nextjs-app.svc.cluster.local:80
```

## Validation Criteria

- ✓ Next.js application builds successfully
- ✓ Docker image created and loaded into Minikube
- ✓ Deployment created with 1 replica running
- ✓ Pod status is Running (1/1 containers ready)
- ✓ Service created with ClusterIP type
- ✓ Health endpoint returns 200 with healthy status
- ✓ Readiness endpoint returns 200 with ok status
- ✓ Home page renders correctly
- ✓ Liveness and readiness probes configured and passing

## Cleanup (Optional)

```bash
# Delete application resources
kubectl delete namespace nextjs-app

# Remove Docker image from Minikube
minikube image rm nextjs-k8s-app:latest

# Remove local files
rm -rf /tmp/nextjs-k8s-app
```

## Notes for Claude Code and Goose Agents

### Execution Strategy
- **Claude Code**: Execute bash commands sequentially using the Bash tool
- **Goose**: Use toolkit.run_shell() for each command block
- Both agents should validate each step before proceeding
- Capture output for validation

### Common Pitfalls
- Node.js build requires sufficient memory (at least 2GB recommended)
- Next.js standalone output is required for minimal container size
- Image must be loaded into Minikube using `minikube image load`
- Port 3000 is the default Next.js port (not configurable without rebuild)
- Health endpoints must return 200 status for probes to pass

### Agent-Specific Hints
- **Wait for deployment**: Use `kubectl wait --for=condition=available` with appropriate timeout
- **Validation**: All curl commands should return HTTP 200 and expected JSON
- **Idempotency**: Use `--dry-run=client -o yaml | kubectl apply -f -` for recreatable resources
- **Debugging**: Check pod logs with `kubectl logs -n nextjs-app -l app=nextjs-k8s-app`
- **Build issues**: If npm install fails, ensure Node.js 18+ is available

### Configuration Notes
- Application runs on port 3000 (Next.js default)
- Service exposes port 80, forwards to container port 3000
- Standalone output mode creates optimized production build
- Resource limits: 512Mi memory, 500m CPU
- Liveness probe: /api/health, 10s initial delay
- Readiness probe: /api/ready, 5s initial delay

### Testing Scenarios

#### Scenario 1: Pod Restart
1. Delete the pod: `kubectl delete pod -n nextjs-app -l app=nextjs-k8s-app`
2. Verify new pod starts automatically
3. Test health endpoints after restart

#### Scenario 2: Service Discovery
1. Deploy a test pod: `kubectl run -it --rm debug --image=alpine --restart=Never -n nextjs-app -- sh`
2. Install curl: `apk add curl`
3. Test service: `curl http://nextjs-k8s-app.nextjs-app.svc.cluster.local/api/health`

#### Scenario 3: Resource Limits
1. Check resource usage: `kubectl top pod -n nextjs-app`
2. Verify pod stays within limits
3. Test under load (optional)

### Observability
- **Logs**: `kubectl logs -n nextjs-app -l app=nextjs-k8s-app --tail=100 -f`
- **Events**: `kubectl get events -n nextjs-app --sort-by='.lastTimestamp'`
- **Pod status**: `kubectl describe pod -n nextjs-app -l app=nextjs-k8s-app`
- **Resource usage**: `kubectl top pod -n nextjs-app`

### Build Optimization Notes
- Multi-stage Dockerfile minimizes final image size
- Standalone output removes unnecessary dependencies
- node_modules not copied to final image
- Static assets optimized during build
- Production build enables React optimizations

## References
- Next.js Documentation: https://nextjs.org/docs
- Next.js Standalone Output: https://nextjs.org/docs/pages/api-reference/next-config-js/output
- Kubernetes Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Docker Multi-Stage Builds: https://docs.docker.com/build/building/multi-stage/
- Next.js in Docker: https://github.com/vercel/next.js/tree/canary/examples/with-docker
