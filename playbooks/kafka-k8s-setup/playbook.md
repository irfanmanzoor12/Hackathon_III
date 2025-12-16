---
title: Kafka K8s Setup
description: Deploy Apache Kafka to Kubernetes using Helm
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - helm
---

# Kafka K8s Setup

## Goal
Deploy Apache Kafka to a Kubernetes cluster (Minikube) using Helm, with verification that the cluster is ready for event-driven applications.

## Prerequisites Check
Before proceeding, verify:
1. Minikube is running
2. kubectl is configured and accessible
3. Helm 3.x is installed

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
```

### 2. Add Bitnami Helm Repository
```bash
# Add Bitnami repo (official Kafka Helm charts)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repo to get latest charts
helm repo update
```

### 3. Create Kafka Namespace
```bash
# Create dedicated namespace for Kafka
kubectl create namespace kafka --dry-run=client -o yaml | kubectl apply -f -
```

### 4. Install Kafka Using Helm
```bash
# Install Kafka with sensible defaults for local development
helm install kafka bitnami/kafka \
  --namespace kafka \
  --set replicaCount=1 \
  --set zookeeper.replicaCount=1 \
  --set listeners.client.protocol=PLAINTEXT \
  --set listeners.controller.protocol=PLAINTEXT \
  --set listeners.interbroker.protocol=PLAINTEXT \
  --set persistence.enabled=false \
  --wait --timeout 5m
```

### 5. Verification & Validation

#### Check Deployment Status
```bash
# List all resources in kafka namespace
kubectl get all -n kafka

# Check pod status (should be Running)
kubectl get pods -n kafka -w

# View Kafka service
kubectl get svc -n kafka
```

#### Verify Kafka Functionality
```bash
# Port-forward Kafka service for local access
kubectl port-forward -n kafka svc/kafka 9092:9092 &

# Create test topic
kubectl exec -it kafka-0 -n kafka -- kafka-topics.sh \
  --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1

# List topics
kubectl exec -it kafka-0 -n kafka -- kafka-topics.sh \
  --list \
  --bootstrap-server localhost:9092

# Produce test message
echo "Hello Kafka from CAPS" | kubectl exec -i kafka-0 -n kafka -- kafka-console-producer.sh \
  --topic test-topic \
  --bootstrap-server localhost:9092

# Consume test message
kubectl exec -it kafka-0 -n kafka -- kafka-console-consumer.sh \
  --topic test-topic \
  --from-beginning \
  --max-messages 1 \
  --bootstrap-server localhost:9092
```

## Validation Criteria
- ✓ Kafka pods in Running state
- ✓ Zookeeper pods in Running state
- ✓ Services created and accessible
- ✓ Test topic created successfully
- ✓ Messages can be produced and consumed

## Cleanup (Optional)
```bash
# Uninstall Kafka
helm uninstall kafka -n kafka

# Delete namespace
kubectl delete namespace kafka
```

## Notes for Claude Code and Goose Agents

### Execution Strategy
- **Claude Code**: Execute all bash commands sequentially using the Bash tool
- **Goose**: Use toolkit.run_shell() for each command block
- Both agents should capture output and validate each step before proceeding

### Common Pitfalls
- If `helm repo add` fails, repo may already exist (safe to continue)
- Port-forward runs in background - kill with `pkill -f "port-forward.*kafka"` if needed
- Interactive consumer (`-it`) may not work in non-TTY environments - use `--timeout-ms 5000` instead

### Agent-Specific Hints
- **Wait for pods**: Use `kubectl wait --for=condition=Ready pod -l app.kubernetes.io/name=kafka -n kafka --timeout=300s`
- **Non-interactive testing**: Replace interactive consumer with: `--timeout-ms 5000 --max-messages 1`
- **Verification**: Check exit codes - all commands should return 0 for success
- **Idempotency**: Playbook can be re-run; existing resources will be detected

### Configuration Notes
- This setup uses persistence.enabled=false for local dev (data is ephemeral)
- For production: enable persistence and increase replicaCount
- Default configuration uses PLAINTEXT protocol (no auth/encryption)
- Kafka will be accessible at: kafka.kafka.svc.cluster.local:9092 from within cluster

## References
- Bitnami Kafka Helm Chart: https://github.com/bitnami/charts/tree/main/bitnami/kafka
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
