# Q67. Create Namespace `limited` with ResourceQuota

## Task
Create namespace `limited` with ResourceQuota:
- CPU requests: 4 cores
- Memory requests: 8Gi
- CPU limits: 8 cores
- Memory limits: 16Gi
- Max pods: 20
- Max services: 10
- Max PVCs: 15

## Solution

```bash
# Step 1: Create namespace
kubectl create namespace limited

# Step 2: Create ResourceQuota (imperative)
kubectl create quota limited-quota \
  --hard=requests.cpu=4,requests.memory=8Gi,limits.cpu=8,limits.memory=16Gi,count/pods=20,count/services=10,count/persistentvolumeclaims=15 \
  --namespace=limited

# Verify
kubectl get quota -n limited
kubectl describe resourcequota limited-quota -n limited
```

## ResourceQuota YAML

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: limited-quota
  namespace: limited
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    count/pods: "20"
    count/services: "10"
    count/persistentvolumeclaims: "15"
```

## ⚠️ Important Side Effect

Once a ResourceQuota sets CPU/Memory limits on a namespace, **every Pod in that namespace MUST declare its own `resources.requests` and `resources.limits`**, or it will be rejected:

```
Error from server (Forbidden): must specify limits.cpu, limits.memory, requests.cpu, requests.memory
```

### Fix Option 1: Add resources to Pod YAML
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Fix Option 2: Create a LimitRange (Best Practice)
A LimitRange auto-injects default resources for Pods that don't specify them.
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: limited
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
```
