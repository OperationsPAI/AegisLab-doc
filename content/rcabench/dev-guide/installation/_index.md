---
title: Installation
weight: 1
---

## 📋 System Requirements

### Hardware Dependencies

**Minimum Requirements**

| Component   | Requirement                        |
| ----------- | ---------------------------------- |
| **CPU**     | 2 cores                            |
| **Memory**  | 4GB RAM                            |
| **Storage** | 10GB available disk space          |
| **OS**      | Linux, macOS, or Windows with WSL2 |

**Recommended Requirements**

| Component   | Requirement                        |
| ----------- | ---------------------------------- |
| **CPU**     | 4+ cores                           |
| **Memory**  | 8GB+ RAM                           |
| **Storage** | 20GB+ available SSD storage        |
| **OS**      | Linux (Ubuntu 20.04+ or CentOS 8+) |

### Software Dependencies

**Required**

- **Docker** (>= 20.10)
- **kubectl** (compatible with your Kubernetes version)
- **Git** (for cloning the repository)

**For Kubernetes Deployment**

- **Kubernetes** (>= 1.25) or **kind/minikube** for local development
- **Helm** (>= 3.0) - optional but recommended

**For Development**

- **Go** (>= 1.23)
- **Python** (>= 3.8)
- **Make** (for using Makefile commands)

---

## 🛠️ Local Development Setup

> [!NOTE]
>
> **Docker Compose** is the easiest way to get started with RCABench for development and testing.


### Step 1: Clone Repository

```bash
git clone https://github.com/OperationsPAI/AegisLab.git
cd AegisLab
```

### Step 2: Start Services

```bash
make local-debug

# This will start:
# - MySQL database
# - Redis cache
# - Jaeger tracing
# - BuildKit daemon
# - RCABench API server
```

### Step 3: Verify Installation

```bash
# Check if all services are running
docker ps

# Access Swagger documentation
open http://localhost:8082/swagger/index.html
```



## ☸️ Kubernetes Deployment

### Prerequisites

Ensure you have a Kubernetes cluster ready:

#### Local Cluster (kind)

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name rcabench

# Set kubectl context
kubectl cluster-info --context kind-rcabench
```

#### Local Cluster (minikube)

```bash
# Install minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --memory=8192 --cpus=4

# Enable required addons
minikube addons enable ingress
```

### Deployment Steps


#### Step 1: Install Dependencies

```bash
devbox shell

make check-prerequisites
```

#### Step 2: Configure Storage

For production deployments, set up persistent storage:

> [!TIP]
> 📖 **Original Tutorial**: This step is based on the [OpenEBS Installation](https://openebs.io/docs/quickstart-guide/installation).

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

#### Step 3: Deploy Application

```bash
# Deploy with default configuration
make run

# Check deployment status
make status
```



---

## ✅ Verification

### Health Checks

> [!NOTE]
>
> Assume the base URL: `http://localhost:8082`

```bash
# Check API health
curl http://localhost:8082/health

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

### Service Verification

```bash
# Check all pods are running
kubectl get pods -n exp

# Check services
kubectl get services -n exp

# Check logs
kubectl logs -f deployment/rcabench -n exp
```

### Functional Testing

```bash
# Test API endpoints
curl http://localhost:8082/api/v1/algorithms

# Test database connection
curl http://localhost:8082/api/v1/datasets

# Test fault injection capabilities
curl -X POST http://localhost:8082/api/v1/injection/test
```

### Python SDK Verification

```bash
# Install Python SDK
cd sdk/python
pip install -e .

# Test SDK
python -c "
from rcabench import RCABenchSDK
sdk = RCABenchSDK('http://localhost:8082')
print('SDK connection successful:', sdk.health_check())
"
```

---

## 🚨 Having Issues?

If you encounter problems during installation or deployment:

{{< cards >}}
{{< card url="../troubleshooting/installation" title="Troubleshooting Guide" subtitle="Common installation issues and solutions" >}}
{{< /cards >}}

> [!IMPORTANT]
>
> **Most installation issues** are related to missing dependencies, insufficient permissions, or resource constraints. Check the troubleshooting guide first!
