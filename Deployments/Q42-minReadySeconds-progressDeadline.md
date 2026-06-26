# Q42. Deployment Stability — minReadySeconds, progressDeadlineSeconds, Rolling Update Strategy

## Task
A Deployment `web-app` is taking traffic before pods are fully stable.
- Ensure new pods are considered available only after **30 seconds** of readiness
- Ensure rollout fails if it takes more than **10 minutes**
- Update rollout strategy: **25% surge** and **25% unavailable**

## Key Fields

```bash
kubectl explain deployment.spec | grep -i seconds
# Fields: maxUnavailable, maxSurge, minReadySeconds, progressDeadlineSeconds
```

## Solution YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  replicas: 4
  minReadySeconds: 30           # Pod must be ready for 30s before considered Available
  progressDeadlineSeconds: 600  # Rollout fails if not complete within 10 minutes
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%       # 1 pod unavailable (25% of 4)
      maxSurge: 25%             # 1 extra pod (25% of 4 = 5 total)
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

## Apply & Watch

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/web-app
kubectl get deployment web-app -w
```

## Notes
- `minReadySeconds: 30` → after readiness probe passes, wait 30s before marking pod Available
- `progressDeadlineSeconds: 600` → marks rollout Failed after 10 minutes
- maxUnavailable=25% with 4 replicas → 1 pod can be down at a time
- maxSurge=25% with 4 replicas → max 5 pods exist simultaneously during rollout
