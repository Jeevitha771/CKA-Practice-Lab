# Q3. Rollout History and Revision Management for api-server

## Task
Check rollout status, inspect history, rollback to a specific revision for `api-server` deployment.

## Solution

```bash
# Check current rollout status
kubectl rollout status deployment/api-server

# View full rollout history
kubectl rollout history deployment/api-server

# Inspect a specific revision
kubectl rollout history deployment/api-server --revision=2

# Rollback to the previous revision
kubectl rollout undo deployment/api-server

# Rollback to a specific revision
kubectl rollout undo deployment/api-server --to-revision=1

# Verify rollout after undo
kubectl rollout status deployment/api-server

# Verify pods
kubectl get pods -l app=api-server
```
