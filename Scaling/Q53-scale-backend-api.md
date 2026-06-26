# Q53. Scale Deployment `backend-api` from 3 to 8 Replicas

## Task
Scale deployment `backend-api` from 3 to 8 replicas.
Verify all pods running, distributed across nodes, service endpoints updated.

## Solution

```bash
# Step 1: Scale to 8 replicas
kubectl scale deployment backend-api --replicas=8

# Step 2: Verify deployment
kubectl get deployment backend-api
# DESIRED=8, READY=8

# Step 3: Verify pods are running and distributed across nodes
kubectl get pods -l app=backend-api -o wide
# Expect: 8 pods, STATUS=Running, each with a NODE assigned

# Step 4: Expose to a Service (if not already)
kubectl expose deployment backend-api \
  --name=backend-service \
  --port=80 \
  --target-port=8080 \
  --type=ClusterIP

# Step 5: Verify service endpoints reflect all 8 pods
kubectl get svc backend-service
kubectl get endpoints backend-service
# Endpoints should list IPs of all 8 pods
```

## Notes
- `kubectl get pods -o wide` shows which node each pod is scheduled on
- `kubectl get endpoints` confirms the service is routing to all new pods
- Pods spread across nodes depends on scheduler and node capacity
