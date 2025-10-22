---
title: RCABench
weight: 3
type: docs
---

This document describes how to deploy RCABench, a fault injection framework for microservice applications.

{{< callout type="info" icon="github">}}
**Source Code:** [OperationsPAI/AegisLab](https://github.com/OperationsPAI/AegisLab/)
{{< /callout >}}

## Prerequisites

Before you begin, ensure you have the following tools installed:

1. **[Helm](https://helm.sh/docs/intro/install/)**: For managing Kubernetes applications.
2. **[Skaffold](https://skaffold.dev/docs/install/)**: For building Docker images.
3. **[Kubectl](https://kubernetes.io/docs/tasks/tools/)**: For interacting with your Kubernetes cluster.
4. **Docker**: For building container images.
5. **Go**: For building RCABench web server.
6. **Make**: For using the provided Makefile to streamline deployment tasks.

{{< callout type="important" >}}
You also need access to a container registry (e.g., Docker Hub, a private registry) to push and pull images.
{{< /callout >}}

## Installation

### Install Kubernetes Job Dependencies

```bash
# Install secrets management for Kubernetes jobs
make install-secrets

# Install hostpath provisioner for local persistent storage
make install-hostpath
```

### Deployment

Use the RCABench Makefile to deploy RCABench components.

{{< tabs items="Local, Kubernetes" >}}

{{< tab >}}

```bash
# Start all services with Docker Compose
make local-debug

# This will start:
# - MySQL database
# - Redis cache
# - Jaeger tracing
# - BuildKit daemon
# - RCABench web server
```

The service endpoints will be accessible at `http://localhost:8082` for the RCABench web server.

{{< /tab >}}

{{< tab >}}

{{< callout type="important" >}}
Ensure you have a Kubernetes cluster ready.
{{< /callout >}}

{{% steps %}}

### Configure Storage

Set up persistent storage for RCABench components.

{{< callout type="info" >}}
**Original Tutorial:** This step is based on the [OpenEBS Installation](https://openebs.io/docs/quickstart-guide/installation).
{{< /callout >}}

```bash
# Add OpenEBS Helm repository
helm repo add openebs https://openebs.github.io/openebs
helm repo update

# Install OpenEBS with custom settings
helm install openebs openebs/openebs --namespace openebs \
  --create-namespace \
  --set localpv-provisioner.global.imageRegistry=docker.1ms.run \
  --set engines.local.lvm.enabled=false \
  --set engines.local.zfs.enabled=false \
  --set engines.local.rawfile.enabled=false \
  --set engines.replicated.mayastor.enabled=false \
  --set loki.enabled=false \
  --set alloy.enabled=false
```

### Deploy RCABench to Kubernetes

```bash
# Check environment dependencies
make check-prerequisites

# Deploy with default configuration
make run
```

{{% /steps %}}

{{< /tab >}}

{{< /tabs >}}

### Verification

{{< tabs items="Local, Kubernetes" >}}

{{< tab >}}

```bash
# Check if all services are running
docker ps

# Check System health
curl http://localhost:8082/system/health

# Expected response:
# {
#   "code": 200,
#   "message": "Success",
#   "data": {
#     "status": "healthy",
#     "timestamp": "2025-09-25T10:30:45Z",
#     "version": "1.0.0",
#     "uptime": "72h30m15s",
#     "services": {
#       "database": {
#         "status": "healthy",
#         "last_checked": "2025-09-25T10:30:45Z",
#         "response_time": "5ms"
#       }
#     }
#   }
# }
```

{{< /tab >}}

{{< tab >}}

```bash
# Check deployment status
make status

# Check logs
kubectl logs deployment/rcabench-consumer -n exp
kubectl logs deployment/rcabench-producer -n exp
```

{{< /tab >}}

{{< /tabs >}}
