# Q41. Implement Canary Deployment

## Task
Implement canary deployment:
- Initial deployment: 3 replicas v1
- Canary: 1 replica v2
- Same labels (so the same Service routes to both)
- Gradually increase canary if stable

## Solution

```bash
# Step 1: Create stable v1 deployment
kubectl create deployment web-app --image=nginx:1.21 --replicas=3
kubectl label deployment web-app app=web tier=frontend

# Step 2: Expose via Service (routes to both stable + canary via shared label)
kubectl expose deployment web-app --name=web-svc --port=80 --target-port=80

# Step 3: Create canary v2 deployment (1 replica)
kubectl create deployment web-app-canary --image=nginx:1.22 --replicas=1
kubectl label deployment web-app-canary app=web tier=frontend

# Step 4: Gradually shift traffic — increase canary
kubectl scale deployment web-app-canary --replicas=2

# Step 5: Scale down stable once canary is proven stable
kubectl scale deployment web-app --replicas=0

# Step 6: Full promotion — canary becomes primary
kubectl scale deployment web-app-canary --replicas=5
kubectl scale deployment web-app-canary --replicas=0
```

## Change Labels on Pods Inside a Deployment (imperative)

```bash
kubectl patch deployment web-app \
  -p '{"spec":{"template":{"metadata":{"labels":{"env":"production"}}}}}'
```

## Notes
- Both deployments share the same label (e.g. `app=web`) so the Service load-balances across all pods
- Traffic split is proportional to replica count:
  - 3 stable + 1 canary = 25% canary traffic
  - 0 stable + 5 canary = 100% canary traffic
