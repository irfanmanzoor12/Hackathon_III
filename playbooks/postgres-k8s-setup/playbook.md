# PostgreSQL K8s Setup

## Goal
Deploy a production-like PostgreSQL instance on a local Kubernetes cluster (Minikube) using Helm, with persistent storage and basic validation.

This database will serve as the state layer for Hackathon III services.

---

## Prerequisites

Verify the following before proceeding:

- Minikube is installed and running
- kubectl is configured to access the cluster
- Helm 3.x is installed

```bash
minikube status
kubectl cluster-info
helm version
