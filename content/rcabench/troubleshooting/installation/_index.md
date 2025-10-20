---
title: Installation Problems
weight: 1
---

## 🗄️ Database Connection Failed

> [!ERROR]
>
> RCABench cannot connect to the database, causing API failures and health check errors.

### Symptoms

- API returns database connection errors
- Health check endpoint fails
- MySQL connection timeouts

### Solutions


**Step 1: Check MySQL Pod Status**

```bash
kubectl get pods -l app=mysql -n exp
```

**Step 2: Examine MySQL Logs**

```bash
kubectl logs deployment/mysql -n exp --tail=50
```

**Step 3: Verify Database Credentials**

```bash
kubectl get secret rcabench-db-secret -o yaml -n exp
```

**Step 4: Test Database Connection**

```bash
kubectl exec -it deployment/mysql -n exp -- mysql -u root -p rcabench
```



---

## ☸️ Pod Scheduling Issues

> [!ERROR]
>
> Kubernetes pods cannot be scheduled due to resource constraints or node issues.

### Symptoms

- Pods stuck in `Pending` state
- Resource allocation errors
- Node affinity conflicts

### Solutions


**Step 1: Check Node Resources**

```bash
kubectl describe nodes
kubectl top nodes  # if metrics-server is available
```

**Step 2: Examine Pod Events**

```bash
kubectl describe pod <pod-name> -n exp
```

**Step 3: Review Resource Requirements**

```bash
kubectl describe deployment rcabench -n exp | grep -A 10 "Requests\|Limits"
```

**Step 4: Free Up Resources (if needed)**

```bash
# Scale down non-essential workloads
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>
```



---

## 📦 Image Pull Errors

> [!ERROR]
>
> Kubernetes cannot pull container images from the registry.

### Symptoms

- `ImagePullBackOff` errors
- `ErrImagePull` status
- Authentication failures

### Solutions


**Step 1: Verify Image Availability**

```bash
# Test image pull manually
docker pull <image-name>:<tag>
```

**Step 2: Check Registry Credentials**

```bash
kubectl get secret rcabench-registry-secret -n exp -o yaml
```

**Step 3: Review Image Pull Policy**

```bash
kubectl get deployment rcabench -o yaml -n exp | grep imagePullPolicy
```

**Step 4: Update Registry Secret (if needed)**

```bash
kubectl create secret docker-registry rcabench-registry-secret \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n exp
```



---

## 🔐 Permission Errors

> [!ERROR]
>
> RBAC permission issues prevent RCABench from accessing Kubernetes resources.

### Symptoms

- `RBAC permission denied` errors
- `Unauthorized access` errors
- API server rejecting requests

### Solutions


**Step 1: Check Current Permissions**

```bash
kubectl auth can-i create pods --namespace=exp
kubectl auth can-i get deployments --namespace=exp
```

**Step 2: Apply RBAC Configuration**

```bash
kubectl apply -f manifests/rbac.yaml
```

**Step 3: Verify Service Account**

```bash
kubectl get serviceaccount rcabench -n exp
kubectl describe serviceaccount rcabench -n exp
```

**Step 4: Check Role Bindings**

```bash
kubectl get rolebindings -n exp
kubectl describe rolebinding rcabench-binding -n exp
```



---

## 🌐 Network Connectivity Issues

> [!ERROR]
>
> Network communication between services fails due to connectivity or DNS issues.

### Symptoms

- Service unreachable errors
- Connection timeouts
- DNS resolution failures

### Solutions


**Step 1: Check Service Endpoints**

```bash
kubectl get endpoints -n exp
kubectl get services -n exp
```

**Step 2: Test Internal Connectivity**

```bash
# Run a debug pod
kubectl run debug-pod --rm -i --tty --image=nicolaka/netshoot -- /bin/bash

# Inside the pod, test DNS resolution
nslookup rcabench-service.exp.svc.cluster.local

# Test HTTP connectivity
curl -v http://rcabench-service.exp.svc.cluster.local:8082/health
```

**Step 3: Check Network Policies**

```bash
kubectl get networkpolicies -n exp
kubectl describe networkpolicy <policy-name> -n exp
```

**Step 4: Verify Ingress Configuration**

```bash
kubectl get ingress -n exp
kubectl describe ingress rcabench-ingress -n exp
```



---

## 🆘 Getting Help

If you encounter issues not covered here:

1. **Check logs**: `make logs` or `kubectl logs -f deployment/rcabench -n exp`
2. **Verify configuration**: Review your `hugo.toml` file
3. **Resource monitoring**: Check CPU/memory usage with `kubectl top pods -n exp`

---

## 🆘 Still Having Issues?

If you're still experiencing problems after trying these solutions:

{{< cards >}}
{{< card url="https://github.com/OperationsPAI/AegisLab/issues" title="Report a Bug" subtitle="Create an issue on GitHub" >}}
{{< /cards >}}

> [!IMPORTANT]
>
> When reporting issues, include your system information, error logs, and steps to reproduce the problem.
