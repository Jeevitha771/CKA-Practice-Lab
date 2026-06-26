# Q1. Create Deployment `web-app`

## Task
Create Deployment `web-app` in namespace `production`:
- Image: nginx:1.21
- 3 replicas
- Labels: app=web / tier=frontend / env=production
- Requests: CPU 100m / Memory 128Mi
- Limits: CPU 200m / Memory 256Mi
- Rolling update: maxUnavailable=1 / maxSurge=1

## Spec Summary
| Field | Value |
|---|---|
| name | web-app |
| namespace | production |
| image | nginx:1.21 |
| replicas | 3 |
| labels | app=web / tier=frontend / env=production |
| CPU request/limit | 100m / 200m |
| Memory request/limit | 128Mi / 256Mi |
| maxUnavailable | 1 |
| maxSurge | 1 |

## Solution

```bash
# Create namespace
kubectl create namespace production

# Create deployment
kubectl create deployment web-app \
  --image=nginx:1.21 \
  --replicas=3 \
  --namespace=production

# Add labels
kubectl label deployment web-app app=web tier=frontend env=production -n production

# Set resource requests and limits
kubectl set resources deployment web-app \
  --limits=cpu=200m,memory=256Mi \
  --requests=cpu=100m,memory=128Mi \
  -n production

# Set rolling update strategy
kubectl patch deployment web-app -n production \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'

# Verify
kubectl get deployment web-app -n production
kubectl describe deployment web-app -n production
```
