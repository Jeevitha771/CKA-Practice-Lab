# Q69. Debug Pod Stuck Pending with "Insufficient CPU"

## Task
Pod stuck `Pending` with "Insufficient cpu". Investigate: node resources, pod requests, ResourceQuota usage, LimitRange constraints. Resolve.

## Reproduce the Problem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100"   # ❌ 100 CPU cores — no node has this!
        memory: "256Mi"
```

```bash
kubectl apply -f broken-cpu-pod.yaml
kubectl get pods
# STATUS: Pending  (not CrashLoopBackOff — just stuck)
```

## Debugging Steps

```bash
# Step 1: Observe the problem
kubectl get pods
# cpu-hog-pod   0/1   Pending   0   2m

# Step 2: Check pod events
kubectl describe pod cpu-hog-pod
# Warning  FailedScheduling  0/1 nodes are available: 1 Insufficient cpu.

# Step 3: Check what the pod is requesting
kubectl describe pod cpu-hog-pod | grep -A5 Requests

# Step 4: Check node available capacity
kubectl describe nodes | grep -A10 "Allocated resources"

# Step 5: Check ResourceQuota (if namespace has one)
kubectl get quota -n default
kubectl describe quota -n default

# Step 6: Check LimitRange (if namespace has one)
kubectl get limitrange -n default
kubectl describe limitrange -n default
```

## Fix

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-hog-pod
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"     # ✅ 100 millicores = 0.1 CPU
        memory: "256Mi"
```

```bash
kubectl delete pod cpu-hog-pod
kubectl apply -f fixed-cpu-pod.yaml
kubectl get pods -w
# Pending → ContainerCreating → Running ✅
```

## Key Concept: Bouncer vs Real Estate Agent

| Scenario | What Happens |
|---|---|
| **With LimitRange/Quota** (Bouncer) | Pod rejected at API Server → never appears in `kubectl get pods` |
| **Without limits** (Real Estate Agent) | Pod created ✅ but Scheduler can't place it → `Pending` indefinitely |

A pod stuck in `Pending` with `Insufficient cpu` = **Scheduler problem** (no LimitRange blocked it at door).

## The Golden Formula

```
Available = Node Capacity - System Reserved - Existing Pod REQUESTS
If Available >= New Pod REQUEST → schedules
Otherwise → Pending
```

## Notes
- Scheduler uses **Requests** for placement (not Limits)
- `1 CPU = 1000m` (millicores); `1Gi = 1024Mi`
- Always convert units before doing capacity math
