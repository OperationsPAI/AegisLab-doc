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

{{< cards cols="1" >}}
{{< card link="../deployment/chaos-mesh" title="Install Chaos Mesh" icon="document" >}}
{{< /cards >}}

### Step 2: Deploy Train Ticket

{{< cards cols="1" >}}
{{< card link="../deployment/train-ticket" title="Deploy Train Ticket" icon="document" >}}
{{< /cards >}}

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
