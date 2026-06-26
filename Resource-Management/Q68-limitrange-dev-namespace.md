# Q68. Create LimitRange in Namespace `dev`

## Task
Create LimitRange in namespace `dev`:
- Default CPU request: 100m / limit: 500m
- Default memory request: 128Mi / limit: 512Mi
- Min CPU: 50m / memory: 64Mi
- Max CPU: 2 / memory: 2Gi

## Solution

```yaml
# 1. Create the 'dev' namespace
apiVersion: v1
kind: Namespace
metadata:
  name: dev

---
# 2. Create the LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-container-limits
  namespace: dev
spec:
  limits:
  - type: Container

    # default = The automatic ceiling injected if container omits limits
    default:
      cpu: "500m"
      memory: "512Mi"

    # defaultRequest = The automatic floor injected if container omits requests
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"

    # max = No single container can exceed this
    max:
      cpu: "2"
      memory: "2Gi"

    # min = No single container can go below this
    min:
      cpu: "50m"
      memory: "64Mi"
```

```bash
kubectl apply -f limitrange.yaml

# Verify
kubectl describe limitrange dev-container-limits -n dev

# Test: create a pod without resources — LimitRange auto-injects defaults
kubectl run test-pod --image=nginx -n dev
kubectl get pod test-pod -n dev -o yaml | grep -A10 resources
```

## The 4 Powers of LimitRange

| Power | Field | Behaviour |
|---|---|---|
| Auto-inject request | `defaultRequest` | Pod forgot to set CPU/memory requests → auto-filled |
| Auto-inject limit | `default` | Pod forgot to set CPU/memory limits → auto-filled |
| Block oversized containers | `max` | Pod requests more than max → rejected |
| Block undersized containers | `min` | Pod requests less than min → rejected |

## ResourceQuota vs LimitRange

| Object | Scope | What It Controls |
|---|---|---|
| `ResourceQuota` | Entire Namespace | Total sum of all pods combined |
| `LimitRange` | Individual Container | Each container's min/max/defaults |

## Notes
- LimitRange and ResourceQuota work best **together** in the same namespace
- LimitRange fixes "lazy developer" pods so ResourceQuota doesn't reject them
- `type: Container` applies rules per container; `type: Pod` applies to the whole pod
