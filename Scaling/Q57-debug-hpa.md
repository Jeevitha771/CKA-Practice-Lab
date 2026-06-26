# Q57. Debug HPA Not Scaling Deployment

## Task
HPA not scaling deployment. Debug: check metrics server, resource requests, HPA status/conditions, current metrics.

## Debugging Steps

```bash
# Step 1: Check HPA status
kubectl get hpa
# Look for: TARGETS showing <unknown> → metrics not available

# Step 2: Describe HPA for detailed conditions
kubectl describe hpa <hpa-name>
# Look for events and conditions like:
#   - "unable to get metrics for resource cpu"
#   - "missing request for cpu"
#   - "FailedGetScale"

# Step 3: Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes   # Should return data; if error → metrics-server issue

# Step 4: Check if deployment has resource requests defined
kubectl get deployment <deploy-name> -o yaml | grep -A6 resources
# resources.requests.cpu MUST be set for CPU-based HPA

# Step 5: Check current metric values
kubectl get hpa <hpa-name> -o yaml
# Look at: status.currentMetrics
```

## Common Causes and Fixes

| Issue | Symptom | Fix |
|---|---|---|
| Metrics server not installed | `kubectl top` fails | Install metrics-server |
| Missing `--kubelet-insecure-tls` | metrics-server CrashLoopBackOff | Add arg to deployment |
| No resource requests on pods | `TARGETS: <unknown>/60%` | Add `resources.requests.cpu` |
| HPA targets wrong deployment | No scaling | Fix `scaleTargetRef.name` |
| Min replicas = current replicas | Never scales down | Check `minReplicas` value |

## Fix: Add Resource Requests

```bash
kubectl set resources deployment web-frontend \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=512Mi
```

## Fix: Metrics Server TLS

```bash
kubectl edit deployment metrics-server -n kube-system
# Add to args:
#   - --kubelet-insecure-tls
```

## Verify After Fix

```bash
kubectl get hpa -w
# TARGETS should now show actual values like: 20%/60%
kubectl describe hpa <hpa-name>
```
