---
title: Fault Injection Problems
weight: 2
---

## 🚫 Fails to Start

> [!ERROR]
>
> Fault injection requests fail to initialize, preventing chaos experiments from running and leaving target services unaffected by intended faults.

### Symptoms

- Injection requests return errors
- Target pods not affected by faults
- Chaos resources not created

### Diagnosis

```bash
# Check chaos resources
kubectl get podchaos -A
kubectl get networkchaos -A
kubectl get stresschaos -A

# Check chaos-mesh operator
kubectl get pods -n chaos-engineering

# Check injection logs
kubectl logs -f -l app=rcabench -n exp | grep injection
```

### Solutions


#### Step 1: Install Chaos Mesh

```bash
# Install chaos-mesh operator
curl -sSL https://mirrors.chaos-mesh.org/v2.6.2/install.sh | bash

# Verify installation
kubectl get pods -n chaos-engineering
```

#### Step 2: RBAC Permissions

```bash
# Grant chaos permissions to service account
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rcabench-chaos
rules:
- apiGroups: ["chaos-mesh.org"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rcabench-chaos
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rcabench-chaos
subjects:
- kind: ServiceAccount
  name: rcabench
  namespace: exp
EOF
```

#### Step 3: Target Pod Selection

```bash
# Check target pods exist
kubectl get pods -l app=target-service -n production

# Verify label selectors
kubectl get pods --show-labels -n production

# Test chaos resource manually
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: test-chaos
  namespace: exp
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: target-service
  duration: "30s"
EOF
```


