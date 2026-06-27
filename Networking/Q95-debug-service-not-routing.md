# Q95. Service Not Routing Traffic — Systematic Debug

## Task
A service exists but traffic is not reaching pods. Debug by checking:
1. Selector matches pod labels
2. Endpoints are populated
3. Pod readiness
4. Actual connectivity

---

## Phase 1: Break the Environment (Lab Setup)

Create a deployment and a broken service (selector mismatch):

```bash
# Deploy with label app=backend
kubectl create deployment backend --image=nginx --replicas=2

# Create a service with WRONG selector (app=backend-api instead of app=backend)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend-api      # WRONG — pods have app=backend
  ports:
  - port: 80
    targetPort: 80
EOF
```

---

## Phase 2: Observe the Symptom

```bash
kubectl run client --image=busybox:1.28 --rm -it -- wget -qO- backend-svc
```

Result:
```
wget: can't connect to remote host (10.96.x.x): Connection refused
# or: wget: can't connect to remote host: Connection timed out
```

---

## Phase 3: Debug Step-by-Step

### Step 1 — Check the service exists and its type

```bash
kubectl get service backend-svc
kubectl describe service backend-svc
```

Look at the `Selector` field in `describe` output:
```
Selector:          app=backend-api
```

### Step 2 — Check endpoints (most important step)

```bash
kubectl get endpoints backend-svc
```

```
NAME          ENDPOINTS   AGE
backend-svc   <none>      30s
```

> ❌ `<none>` means no pod matched the selector. This is the root cause indicator.

### Step 3 — Check actual pod labels

```bash
kubectl get pods --show-labels -l app=backend
```

```
NAME                       READY   STATUS    LABELS
backend-7d9f8b-abc         1/1     Running   app=backend,pod-template-hash=7d9f8b
backend-7d9f8b-xyz         1/1     Running   app=backend,pod-template-hash=7d9f8b
```

> Pods have `app=backend` but the service selector requires `app=backend-api`.
> **Mismatch confirmed.**

### Step 4 — Check pod readiness

Even if the selector matches, pods must be **Ready** for the service to include them.

```bash
kubectl get pods -l app=backend
```

```
NAME                   READY   STATUS    RESTARTS   AGE
backend-7d9f8b-abc     1/1     Running   0          2m
backend-7d9f8b-xyz     1/1     Running   0          2m
```

> `READY: 1/1` — pods are ready. If `0/1`, check readiness probe:
> ```bash
> kubectl describe pod <pod-name> | grep -A 10 Readiness
> ```

### Step 5 — Test direct pod connectivity (bypass service)

```bash
# Get a pod IP
POD_IP=$(kubectl get pod -l app=backend -o jsonpath='{.items[0].status.podIP}')

# Test direct pod access (bypasses service completely)
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- $POD_IP:80
```

> If direct pod access works but service doesn't → selector mismatch or kube-proxy issue.
> If direct pod access also fails → problem is with the pod/app itself.

---

## Phase 4: Fix the Issue

### Fix 1 — Correct the selector (most common fix)

```bash
kubectl patch service backend-svc \
  -p '{"spec":{"selector":{"app":"backend"}}}'
```

Or edit directly:

```bash
kubectl edit service backend-svc
# Change: app: backend-api
# To:     app: backend
```

### Fix 2 — Add missing label to pods (alternative approach)

```bash
kubectl label pods -l app=backend app=backend-api
```

> Use this only if you want to keep the service selector and add the label to pods instead.

---

## Phase 5: Verify the Fix

```bash
# Endpoints should now be populated
kubectl get endpoints backend-svc
```

```
NAME          ENDPOINTS                             AGE
backend-svc   10.244.0.5:80,10.244.1.3:80           30s
```

```bash
# Traffic should now work
kubectl run client --image=busybox:1.28 --rm -it -- wget -qO- backend-svc
```

```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
```

✅ Service is now routing traffic correctly.

---

## Debug Checklist

| Check | Command | What to Look For |
|---|---|---|
| Service exists | `kubectl get svc <name>` | Status and ClusterIP |
| Selector | `kubectl describe svc <name>` | `Selector:` field |
| Endpoints populated | `kubectl get endpoints <name>` | Not `<none>` |
| Pod labels | `kubectl get pods --show-labels` | Labels match selector |
| Pod readiness | `kubectl get pods` | `READY: 1/1` |
| Pod readiness probe | `kubectl describe pod <name>` | Readiness probe result |
| Direct pod access | `kubectl exec` / `wget <pod-ip>` | Works without service |
| kube-proxy rules | `kubectl get pods -n kube-system` | kube-proxy pods running |

---

## Common Causes Summary

| Root Cause | Symptom | Fix |
|---|---|---|
| Selector mismatch | Endpoints `<none>` | Fix selector or pod label |
| Pod not Ready | Endpoint exists but no traffic | Fix readiness probe / app |
| Wrong targetPort | Timeout at pod | Match targetPort to container port |
| No pods running | Endpoints `<none>` | Scale up deployment |
| Network policy blocking | Endpoints exist but traffic dropped | Add NetworkPolicy allow rule |

---

## Cleanup

```bash
kubectl delete service backend-svc
kubectl delete deployment backend
```
