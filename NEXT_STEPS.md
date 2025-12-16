# Hackathon III - Next Steps

## Overview
This document provides actionable instructions for completing the remaining CAPS playbooks for Hackathon III: Reusable Intelligence and Cloud-Native Mastery.

**Last Updated**: 2025-12-16

---

## Completed Playbooks ✓
- [x] `agents-md-gen` - Generate AGENTS.md for repository guidance
- [x] `kafka-k8s-setup` - Deploy Kafka to Kubernetes using Helm

---

## Remaining Playbooks - Priority Order

### 1. `postgres-k8s-setup` (HIGH PRIORITY)
**Goal**: Deploy PostgreSQL to Kubernetes using Helm for persistent data storage

**Location**: `playbooks/postgres-k8s-setup/playbook.md`

**Required Sections**:
```yaml
---
title: PostgreSQL K8s Setup
description: Deploy PostgreSQL to Kubernetes using Helm
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - helm
---
```

**Key Implementation Steps**:
1. Prerequisites: Minikube running, kubectl, Helm 3.x
2. Add Bitnami Helm repository (if not already added)
3. Create `postgres` namespace
4. Install PostgreSQL with:
   - Single replica for local dev
   - Persistence enabled (PVC)
   - Default credentials via Secret
5. Validation:
   - Pod status check
   - Database connection test
   - Create test table and insert data
   - Query test data

**Agent Instructions**:
- Use `bitnami/postgresql` Helm chart
- Set `auth.postgresPassword` and `auth.database` for testing
- Provide `psql` commands for validation
- Include connection string format for applications

---

### 2. `fastapi-dapr-agent` (HIGH PRIORITY)
**Goal**: Deploy a FastAPI microservice with Dapr sidecar for event-driven communication

**Location**: `playbooks/fastapi-dapr-agent/playbook.md`

**Required Sections**:
```yaml
---
title: FastAPI Dapr Agent
description: Deploy FastAPI microservice with Dapr runtime
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
  - dapr
---
```

**Key Implementation Steps**:
1. Prerequisites: Dapr CLI installed, Dapr initialized on K8s
2. Create FastAPI service:
   - Sample endpoint: `/health`, `/api/message`
   - Dockerfile for containerization
   - Dapr pub/sub integration
3. Dapr configuration:
   - Component YAML for Kafka pub/sub
   - State store (Redis or Postgres)
4. Deploy to Kubernetes:
   - Deployment with Dapr annotations
   - Service exposure
5. Validation:
   - Test HTTP endpoints
   - Publish message to Kafka via Dapr
   - Verify Dapr sidecar logs

**Agent Instructions**:
- Generate complete FastAPI app code
- Include `requirements.txt` and `Dockerfile`
- Provide Dapr component YAMLs
- Use `dapr run` for local testing first

---

### 3. `nextjs-k8s-deploy` (MEDIUM PRIORITY)
**Goal**: Deploy Next.js application to Kubernetes

**Location**: `playbooks/nextjs-k8s-deploy/playbook.md`

**Required Sections**:
```yaml
---
title: Next.js K8s Deployment
description: Deploy Next.js application to Kubernetes
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
  - npm
---
```

**Key Implementation Steps**:
1. Prerequisites: Node.js, Docker, kubectl
2. Create Next.js app (or use existing)
3. Create production Dockerfile:
   - Multi-stage build
   - Optimized for production
4. Build and push image
5. Create Kubernetes manifests:
   - Deployment
   - Service (LoadBalancer or NodePort)
   - Optional: Ingress
6. Deploy to Minikube
7. Validation:
   - Access via `minikube service` or port-forward
   - Verify SSR/SSG functionality

**Agent Instructions**:
- Use `next build` and `next start` for production
- Expose on port 3000
- Include environment variable handling
- Provide scaling instructions

---

### 4. `mcp-python-server` (MEDIUM PRIORITY)
**Goal**: Create Model Context Protocol (MCP) server in Python

**Location**: `playbooks/mcp-python-server/playbook.md`

**Required Sections**:
```yaml
---
title: MCP Python Server
description: Create and deploy MCP server using Python
version: 1.0.0
allowed_tools:
  - bash
  - python
  - pip
  - docker
---
```

**Key Implementation Steps**:
1. Prerequisites: Python 3.10+, pip, FastAPI
2. Create MCP server structure:
   - Server implementation
   - Tool/resource definitions
   - Protocol handlers
3. Local testing with stdio transport
4. Containerize for deployment
5. Integration with Claude Desktop/Code
6. Validation:
   - Test tool invocations
   - Verify resource access
   - Check error handling

**Agent Instructions**:
- Use `mcp` Python SDK
- Provide example tools (file access, computation)
- Include configuration for Claude Desktop
- Document transport options (stdio, HTTP)

---

### 5. `code-runner-service` (LOW PRIORITY)
**Goal**: Deploy code execution service with sandboxing

**Location**: `playbooks/code-runner-service/playbook.md`

**Required Sections**:
```yaml
---
title: Code Runner Service
description: Deploy sandboxed code execution service
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
  - python
---
```

**Key Implementation Steps**:
1. Prerequisites: Docker, Kubernetes
2. Create code runner API:
   - Accept code snippets
   - Execute in sandbox (Docker/gVisor)
   - Return output/errors
3. Security measures:
   - Resource limits (CPU, memory, time)
   - Network isolation
   - Filesystem restrictions
4. Deploy to K8s with security context
5. Validation:
   - Execute test code
   - Verify sandbox isolation
   - Test resource limits

**Agent Instructions**:
- Use Docker-in-Docker or gVisor for sandboxing
- Implement timeout mechanisms
- Support multiple languages (Python, Node.js, Go)
- Include rate limiting

---

### 6. `docusaurus-deploy` (LOW PRIORITY)
**Goal**: Deploy Docusaurus documentation site to Kubernetes

**Location**: `playbooks/docusaurus-deploy/playbook.md`

**Required Sections**:
```yaml
---
title: Docusaurus Deployment
description: Deploy Docusaurus documentation to Kubernetes
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
  - npm
---
```

**Key Implementation Steps**:
1. Prerequisites: Node.js, Docker, kubectl
2. Create/use Docusaurus site
3. Build static site: `npm run build`
4. Create Dockerfile (nginx serving static files)
5. Build and push image
6. Create K8s manifests:
   - Deployment (nginx)
   - Service
   - Optional: Ingress
7. Deploy to Minikube
8. Validation:
   - Access via browser
   - Verify all pages load
   - Test search functionality

**Agent Instructions**:
- Use nginx:alpine for serving
- Configure nginx for SPA routing
- Include health check endpoint
- Provide update/rollback instructions

---

## General Agent Workflow

For each playbook, agents should follow this process:

### Step 1: Initialize
```bash
# Read current status
cat PROJECT_STATUS.md

# Create playbook directory
mkdir -p playbooks/<playbook-name>
```

### Step 2: Create Playbook
```bash
# Create playbook.md with proper CAPS frontmatter
# Include all required sections:
# - Goal
# - Prerequisites
# - Instructions (step-by-step)
# - Validation
# - Notes for Claude Code and Goose
```

### Step 3: Test Execution (Optional but Recommended)
```bash
# Follow playbook instructions
# Capture any issues or improvements
# Update playbook with lessons learned
```

### Step 4: Update Status
```bash
# Move playbook from "Not Started" to "Completed" in PROJECT_STATUS.md
# Update NEXT_STEPS.md if needed
```

---

## Validation Checklist for Each Playbook

Before marking a playbook as complete, verify:

- [ ] CAPS frontmatter is complete (title, description, version, allowed_tools)
- [ ] Goal section clearly states the objective
- [ ] Prerequisites are explicitly listed
- [ ] Instructions are step-by-step and copy-paste ready
- [ ] All bash commands are tested and working
- [ ] Validation section includes verification commands
- [ ] Notes for agents address common pitfalls
- [ ] Playbook works for both Claude Code and Goose
- [ ] Follows AAIF principles (autonomous, single-prompt executable)
- [ ] No hardcoded secrets or credentials
- [ ] Cleanup instructions provided (optional section)

---

## Agent-Specific Execution Notes

### For Claude Code
- Use `Bash` tool for command execution
- Use `Write` tool for file creation
- Use `Edit` tool for file modifications
- Capture and validate command outputs
- Use `Read` tool to verify created files

### For Goose
- Use `toolkit.run_shell()` for bash commands
- Use `toolkit.write_file()` for file creation
- Use `toolkit.read_file()` for verification
- Check return codes for all commands
- Handle interactive prompts appropriately

---

## Success Criteria for Hackathon III

A playbook is considered successful when:

1. **Autonomous**: Single prompt execution from start to finish
2. **Vendor-Neutral**: Works with both Claude Code and Goose
3. **Repeatable**: Can be run multiple times idempotently
4. **Validated**: Includes verification commands that prove success
5. **Documented**: Clear instructions with agent-specific hints
6. **Production-Ready**: Includes notes for production deployment

---

## Quick Reference: Current State

**Completed**: 2/8 playbooks
**In Progress**: 0/8 playbooks
**Not Started**: 6/8 playbooks

**Next Recommended Action**: Create `postgres-k8s-setup` playbook

---

## Questions or Issues?

If agents encounter issues:
1. Check PROJECT_STATUS.md for blockers
2. Review DECISIONS.md for architectural guidance
3. Consult AGENTS.md for operating rules
4. Read .specify/memory/constitution.md for standards

**Note**: All playbooks should optimize for maximum autonomy per Decision 004.
