---
title: Getting Started
date: 2025-10-17
weight: 2
---

This document describes how to start RCABench quickly·

## 📋 Prerequisites

### Hardware Requirements

| Component   | Requirement                        |
| ----------- | ---------------------------------- |
| **CPU**     | 4+ cores                           |
| **Memory**  | 8GB+ RAM                           |
| **Storage** | 20GB+ available SSD storage        |
| **OS**      | Linux (Ubuntu 20.04+ or CentOS 8+) |

### Software Requirements

- **Docker** (>= 20.10)
- **Kubernetes** (>= 1.25) or **kind/minikube** for local development
- **kubectl** (compatible with your cluster version)
- **Helm** (>= 3.0) for Kubernetes deployments
- **Devbox** for local development
- **Skaffold** for building and deploying services
- **Go** (>= 1.23) for development
- **Python** (>= 3.10) for SDK usage

---

## 🚀 Quick Start

### Step 1: Install Chaos Mesh

> [!TIP]
> 📖 **Original Tutorial**: This step is based on the [Chaos Mesh Installation](https://chaos-mesh.org/docs/production-installation-using-helm).

```bash
# Add the Chaos Mesh repository to the Helm repository
helm repo add chaos-mesh https://charts.chaos-mesh.org

# To see charts that can be installed
helm search repo chaos-mesh

# Install Chaos Mesh under the chaos-mesh namespace
kubectl create ns chaos-mesh

helm install chaos-mesh chaos-mesh/chaos-mesh -n=chaos-mesh
    --set chaosDaemon.runtime=containerd \
    --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
    --version 2.7.2
```

#### Access Chaos Mesh Dashboard

```bash
kubectl get svc -n chaos-mesh
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                 AGE
chaos-daemon                    ClusterIP   None            <none>        31767/TCP,31766/TCP                     30m
chaos-dashboard                 NodePort    10.97.177.177   <none>        2333:31632/TCP,2334:32683/TCP           30m
chaos-mesh-controller-manager   ClusterIP   10.105.196.18   <none>        443/TCP,10081/TCP,10082/TCP,10080/TCP   30m
chaos-mesh-dns-server           ClusterIP   10.96.127.183   <none>        53/UDP,53/TCP,9153/TCP,9288/TCP         30m
```

Visit `http://<your-ip>:<your-node-port>` in your browser to access the Chaos Mesh dashboard.

#### Generate Dashboard Token

1. Save below content as `rbac.yaml`:

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  namespace: default
  name: account-cluster

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role-cluster
rules:
  - apiGroups: [""]
    resources: ["pods", "namespaces"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["chaos-mesh.org"]
    resources: ["*"]
    verbs: ["get", "list", "watch", "create", "delete", "patch", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-cluster
subjects:
  - kind: ServiceAccount
    name: account-cluster
    namespace: default
roleRef:
  kind: ClusterRole
  name: role-cluster
  apiGroup: rbac.authorization.k8s.io
```

2. Then run the following command to create the service account and role binding:

```bash
kubectl apply -f rbac.yaml
```

3.  Generate a service account token:

```bash
kubectl create token account-cluster -n default
```

Copy the generated token and use it to log in to the dashboard. The name of the front end can be written as you like.

### Step 2: Deploy Benchmark Application

Deploy a pedestal microservices application `train-ticket` to be used for fault injection and root cause analysis.

```bash
# Create a namespace for the benchmark application
kubectl create namespace ts0

# Deploy the train-ticket application
# ts is the prefix for the namespace and 1 is the number of pedestal instances
make install-releases PEDESTAL_KEY=ts PEDESTAL_COUNT=1
```

### Step 3: Local Development Setup

#### Clone Repository

```bash
git clone https://github.com/OperationsPAI/AegisLab.git
cd AegisLab
```

#### Install Kubernetes Job Dependencies

```bash
make install-secrets

make install-hostpath
```

#### Start Services

```bash
# Start all services with Docker Compose
make local-debug

# This will start:
# - MySQL database
# - Redis cache
# - Jaeger tracing
# - BuildKit daemon
# - AegisLab API server
```

#### Verify Installation

```bash
# Check if all services are running
docker ps

# Test API access
curl http://localhost:8082/system/health

# Access Swagger documentation
open http://localhost:8082/swagger/index.html
```

### Step 2: Setup Python Environment

```bash
# Install RCABench SDK
pip install rcabench

# Or install from source
cd sdk/python
pip install -e .
```

### Step 3: Connect to RCABench

```python
from rcabench import RCABenchSDK
import time

# Initialize SDK
sdk = RCABenchSDK("http://localhost:8082")

# Verify connection
health = sdk.health_check()
print(f"RCABench status: {health}")
```

### Step 4: List Available Resources

```python
# List available algorithms
algorithms = sdk.algorithm.list()
print("Available algorithms:")
for algo in algorithms:
    print(f"  - {algo['name']}: {algo['description']}")

# List available datasets
datasets = sdk.dataset.list()
print("Available datasets:")
for dataset in datasets:
    print(f"  - {dataset['name']}: {dataset['description']}")
```

### Step 5: Inject a CPU Stress Fault

```python
# Define fault injection request
fault_request = DtoSubmitInjectionReq(
    algorithms=[
        DtoAlgorithmItem(name="traceback"),
    ], # Algorithm execution container name
    benchmark="clickhouse", # Data collection container name
    container_name="ts_cn", # Pedestal container name
    container_tag="v1.0.0-213-gf9294111", # Pedestal container tag
    interval=2, # The whole period in minutes
    project_name="pair_diagnosis",
    pre_duration=1, # Normal time in minutes
    specs=[
        HandlerNode(
            children={
                "4": HandlerNode(
                    children={
                        "0": HandlerNode(value=1), # Duration in minutes
                        "1": HandlerNode(value=0), # Namespace prefix ts is to 0
                        "2": HandlerNode(value=1), # Container Index
                        "3": HandlerNode(value=1), # CPU Load Percentage
                        "4": HandlerNode(value=1), # CPU Stress Threads
                    },
                )
            },
            value=4,
        )
    ],
    labels=[
        DtoLabelItem(key="env", value=env),
        DtoLabelItem(key="batch", value="bootstrap"),
    ],
)

# Execute fault injection
injection_result = sdk.injection.execute(fault_request)
```

---

## 📍 Next

Let's install the whole RCABench system:

{{< cards >}}
{{< card url="../guide/installation" title="Installation" >}}
{{< /cards >}}

```

```
