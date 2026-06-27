# Q102. Ingress Returning 404 Errors — Systematic Debug

## Task
An Ingress resource exists but returns 404 for all requests. Debug:
1. Check controller logs
2. Verify ingress config
3. Check backend service and pods
4. Test DNS
5. Verify ingress class

---

## Phase 1: Break the Environment (Lab Setup)

```bash
# Deploy backend
kubectl create deployment myapp --image=nginx
kubectl expose deployment myapp --name=myapp-svc --port=80

# Create a BROKEN Ingress (wrong service name + wrong ingressClassName)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-wrong-svc   # WRONG: service doesn't exist
            port:
              number: 80
EOF
```

---

## Phase 2: Observe the Symptom

```bash
HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

curl -H "Host: myapp.example.com" http://$NODE_IP:$HTTP_PORT
```

```html
<html>
<head><title>404 Not Found</title></head>
<body>nginx 404 Not Found</body>
</html>
```

---

## Phase 3: Debug Step-by-Step

### Step 1 — Check Ingress controller logs

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50
```

Look for:
```
"Ignoring ingress because of error while validating ingress class"
"Service "myapp-wrong-svc" not found"
"Error obtaining endpoints for service"
```

> Controller logs are the **first place** to check — they tell you exactly why routing fails.

### Step 2 — Verify the Ingress resource config

```bash
kubectl get ingress broken-ingress
kubectl describe ingress broken-ingress
```

In `describe` output, check:
```
Rules:
  Host               Path  Backends
  myapp.example.com  /     myapp-wrong-svc:80 (<error: endpoints "myapp-wrong-svc" not found>)
```

> Kubernetes shows `<error: endpoints "..." not found>` inline when the backend service is missing.

### Step 3 — Check the ingressClassName

```bash
kubectl get ingress broken-ingress -o jsonpath='{.spec.ingressClassName}'
```

```
nginx
```

```bash
# Verify the IngressClass actually exists
kubectl get ingressclass
```

```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       10m
```

> If no IngressClass exists, the controller ignores all Ingress resources.

### Step 4 — Check backend service exists

```bash
kubectl get service myapp-wrong-svc
```

```
Error from server (NotFound): services "myapp-wrong-svc" not found
```

> ❌ Root cause confirmed — the service referenced in the Ingress doesn't exist.

```bash
# Find the actual service name
kubectl get services
```

```
NAME         TYPE        CLUSTER-IP   PORT(S)   AGE
myapp-svc    ClusterIP   10.96.x.x    80/TCP    5m
```

### Step 5 — Check backend pods are running and ready

```bash
kubectl get pods -l app=myapp
```

```
NAME                    READY   STATUS    RESTARTS
myapp-xxxxxx-yyy        1/1     Running   0
```

### Step 6 — Check endpoints exist for the correct service

```bash
kubectl get endpoints myapp-svc
```

```
NAME        ENDPOINTS          AGE
myapp-svc   10.244.x.x:80      5m
```

---

## Phase 4: Fix the Issue

```bash
kubectl edit ingress broken-ingress
```

Change:
```yaml
name: myapp-wrong-svc   # wrong
```
To:
```yaml
name: myapp-svc         # correct
```

---

## Phase 5: Verify the Fix

```bash
kubectl describe ingress broken-ingress
```

```
Rules:
  Host               Path  Backends
  myapp.example.com  /     myapp-svc:80 (10.244.x.x:80)
```

> Endpoints are now shown inline — routing is working.

```bash
curl -H "Host: myapp.example.com" http://$NODE_IP:$HTTP_PORT
```

```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
```

✅ 200 OK — Ingress routing is fixed.

---

## Debug Checklist

| Check | Command | What to Look For |
|---|---|---|
| Controller running | `kubectl get pods -n ingress-nginx` | `Running` status |
| Controller logs | `kubectl logs -n ingress-nginx -l app...` | Error messages |
| IngressClass exists | `kubectl get ingressclass` | `nginx` present |
| Ingress config | `kubectl describe ingress <name>` | Backend shows endpoints |
| Service exists | `kubectl get svc <name>` | Not NotFound |
| Endpoints populated | `kubectl get endpoints <name>` | Not `<none>` |
| Pods running/ready | `kubectl get pods -l <selector>` | `1/1 Running` |
| DNS resolves | `nslookup <host>` / curl with Host header | Correct IP |

---

## Common 404 Causes

| Cause | Symptom | Fix |
|---|---|---|
| Wrong service name in Ingress | `<error: endpoints not found>` | Fix `backend.service.name` |
| Wrong IngressClass | Controller ignores the resource | Add/fix `ingressClassName` |
| Wrong path | Route exists but path doesn't match | Fix `path` or use `Prefix` |
| Pods not Ready | Endpoints empty | Fix readiness probe |
| Missing Host header | Test hitting wrong virtual host | Use `-H "Host: ..."` in curl |

---

## Cleanup

```bash
kubectl delete ingress broken-ingress
kubectl delete service myapp-svc
kubectl delete deployment myapp
```
