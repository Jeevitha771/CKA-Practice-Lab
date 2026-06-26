# Q2. Update web-app Deployment, Monitor Rollout, Rollback if Issues

## Task
Update `web-app` deployment to nginx:1.22, record change, monitor rollout, verify all pods updated.
If issues occur, rollback.

## Solution

```bash
# Update image and record change
kubectl set image deployment/web-app nginx=nginx:1.22 --record

# Monitor rollout progress
kubectl rollout status deployment/web-app

# Check rollout history
kubectl rollout history deployment/web-app

# If issues occur — rollback to previous version
kubectl rollout undo deployment/web-app
```

## Notes
- `--record` saves the command in revision history (deprecated in newer k8s; use annotations instead)
- `kubectl rollout status` blocks until rollout completes or fails
- `kubectl rollout undo` reverts to the immediately previous revision
