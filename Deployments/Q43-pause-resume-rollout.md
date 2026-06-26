# Q43. Pause Deployment, Make Multiple Changes, Resume — Single Rollout

## Task
Pause deployment rollout, make multiple changes (image, env vars, resources), resume rollout.
Verify single rollout for all changes.

## Solution

```bash
# Step 1: Pause the rollout
kubectl rollout pause deployment web-app

# Step 2: Make multiple changes (none trigger a rollout while paused)
kubectl set image deployment/web-app nginx=nginx:1.23
kubectl set env deployment/web-app ENV=prod
kubectl set resources deployment web-app \
  --limits=cpu=500m,memory=512Mi \
  --requests=cpu=200m,memory=256Mi

# Optional: patch labels on pod template
kubectl patch deployment web-app \
  -p '{"spec":{"template":{"metadata":{"labels":{"env":"production"}}}}}'

# Step 3: Resume — all changes deploy as ONE rollout
kubectl rollout resume deployment web-app

# Step 4: Monitor
kubectl rollout status deployment web-app

# Step 5: Verify single revision was added
kubectl rollout history deployment web-app
```

## Notes
- Without pause, each `set image`, `set env`, `set resources` triggers a separate rollout
- Pausing batches all changes into a single revision — cleaner history and fewer disruptions
