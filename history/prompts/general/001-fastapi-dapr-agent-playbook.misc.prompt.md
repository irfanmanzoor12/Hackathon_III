---
id: 001
title: Create FastAPI Dapr Agent CAPS Playbook
stage: misc
date: 2025-12-17
surface: agent
model: claude-sonnet-4-5
feature: none
branch: master
user: irfan
command: none
labels: ["caps-playbook", "fastapi", "dapr", "kafka", "postgresql", "kubernetes"]
links:
  spec: null
  ticket: null
  adr: null
  pr: null
files:
  - playbooks/fastapi-dapr-agent/playbook.md
  - PROJECT_STATUS.md
tests:
  - Validation script included in playbook for deployment verification
---

## Prompt

Read AGENTS.md and PROJECT_STATUS.md.
Create CAPS playbook: fastapi-dapr-agent.
Requirements:
- FastAPI service
- Dapr sidecar
- Kafka pub/sub integration
- PostgreSQL connection
- Health and readiness endpoints
- Validation commands

## Response snapshot

Created comprehensive CAPS playbook at playbooks/fastapi-dapr-agent/playbook.md with:

1. Complete FastAPI application with:
   - Health and readiness endpoints with dependency checks
   - Dapr pub/sub integration for Kafka messaging
   - Dapr state management with PostgreSQL backend
   - Direct PostgreSQL connection with connection pooling
   - Comprehensive error handling and logging

2. Full Kubernetes deployment configuration:
   - Deployment with Dapr sidecar annotations
   - Service definition
   - Secret management for PostgreSQL credentials
   - Resource limits and probes

3. Dapr components configuration:
   - Kafka pubsub component
   - PostgreSQL state store component

4. Docker containerization:
   - Multi-stage Dockerfile
   - Requirements.txt with all dependencies
   - Optimized for production

5. Complete validation suite:
   - Health and readiness checks
   - Database connectivity tests
   - State management operations (save/get/delete)
   - Pub/sub functionality tests
   - Comprehensive bash validation script

6. Agent execution notes:
   - Claude Code and Goose compatibility guidance
   - Common pitfalls and solutions
   - Testing scenarios
   - Production considerations

## Outcome

- ✅ Impact: Created vendor-neutral CAPS playbook for deploying FastAPI microservices with Dapr on Kubernetes. Enables single-prompt deployment of cloud-native Python agents with event-driven architecture.
- 🧪 Tests: Comprehensive validation commands included covering health checks, database connectivity, state management, and pub/sub functionality.
- 📁 Files: Created playbook.md (850+ lines) with complete deployment instructions and updated PROJECT_STATUS.md to track completion.
- 🔁 Next prompts: Execute the playbook to deploy and validate the service, or create similar playbooks for nextjs-k8s-deploy, mcp-python-server, or other services.
- 🧠 Reflection: Followed CAPS playbook structure from kafka-k8s-setup as reference. Included extensive agent-specific hints for autonomous execution. Playbook is idempotent and includes cleanup instructions.

## Evaluation notes (flywheel)

- Failure modes observed: None - playbook creation completed successfully
- Graders run and results (PASS/FAIL): N/A - playbook not yet executed
- Prompt variant (if applicable): N/A
- Next experiment (smallest change to try): Execute playbook to validate deployment and identify any missing configuration or edge cases
