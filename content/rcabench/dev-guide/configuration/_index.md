---
title: Configuration
weight: 2
---

## 🌐 Environment Configuration

RCABench uses TOML configuration files. The main configuration sections are:

### 🗄️ Database Configuration

```toml
[database]
mysql_user = "root"
mysql_password = "yourpassword"
mysql_host = "mysql-service"  # Kubernetes service name
mysql_port = "3306"
mysql_db = "rcabench"
```

### 🔄 Redis Configuration

```toml
[redis]
host = "redis-service:6379"  # Kubernetes service name
```

### ☸️ Kubernetes Configuration

```toml
[k8s]
namespace = "exp"  # Target namespace for experiments
```

### 🔧 Injection Configuration

```toml
[injection]
benchmark = ["workload1", "workload2"]
target_label_key = "app"

[injection.namespace_config.exp]
count = 5
port = "80%02d"
```

### ⚡ Rate Limiting

```toml
[rate_limiting]
max_concurrent_builds = 3
max_concurrent_restarts = 2
max_concurrent_algo_execution = 5
token_wait_timeout = 10
```

### 🔐 Secret Management

For production deployments, use Kubernetes secrets:

```bash
# Create database secret
kubectl create secret generic rcabench-db-secret \
  --from-literal=password=yourpassword \
  -n exp

# Create registry secret (if using private registry)
kubectl create secret docker-registry rcabench-registry-secret \
  --docker-server=your-registry.com \
  --docker-username=username \
  --docker-password=password \
  -n exp
```
