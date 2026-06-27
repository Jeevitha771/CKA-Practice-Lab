# Q104. Ingress with Default Backend for Custom 404 Page

## Task
Create an Ingress with a default backend that serves a custom 404 page
when no routing rules match the incoming request.

> **SKIM note:** Know the `defaultBackend` field location and structure.
> This is an edge case but can appear as a quick task on the exam.

---

## Phase 1: Create the Custom 404 Backend

```bash
# Create a custom 404 deployment (serves a custom error page)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-404
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-404
  template:
    metadata:
      labels:
        app: custom-404
    spec:
      containers:
      - name: nginx
        image: nginx
        # In a real scenario you'd mount a ConfigMap with a custom 404 HTML page
EOF
```

```bash
kubectl expose deployment custom-404 --name=custom-404-svc --port=80
```

---

## Phase 2: Create the Main Application

```bash
kubectl create deployment mainapp --image=nginx
kubectl expose deployment mainapp --name=mainapp-svc --port=80
```

---

## Phase 3: Create Ingress with Default Backend

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  ingressClassName: nginx
  # Default backend: serves when NO rules match
  defaultBackend:
    service:
      name: custom-404-svc
      port:
        number: 80
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mainapp-svc
            port:
              number: 80
EOF
```

**Key field:**

| Field | Purpose |
|---|---|
| `spec.defaultBackend` | Catches all requests that don't match any rule |
| `spec.rules` | Normal routing rules (matched first) |

**Traffic flow:**
```
Request: Host=app.example.com, path=/   → mainapp-svc:80
Request: Host=unknown.com, path=/       → custom-404-svc:80 (default backend)
Request: Host=app.example.com, path=/xyz → custom-404-svc:80 (no matching path rule)
```

---

## Phase 4: Verify

```bash
kubectl describe ingress default-backend-ingress
```

```
Default backend: custom-404-svc:80 (10.244.x.x:80)
Rules:
  Host             Path  Backends
  app.example.com  /     mainapp-svc:80
```

---

## Phase 5: Test

```bash
HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

# Known host/path → mainapp
curl -H "Host: app.example.com" http://$NODE_IP:$HTTP_PORT/
# → 200 OK from mainapp

# Unknown host → default backend (custom 404)
curl -H "Host: unknown.example.com" http://$NODE_IP:$HTTP_PORT/
# → 200 OK from custom-404 backend (serving custom error page)
```

---

## Summary: Quick Reference

```bash
# defaultBackend sits at spec level (not under rules)
spec:
  defaultBackend:
    service:
      name: custom-404-svc
      port:
        number: 80
```

---

## Cleanup

```bash
kubectl delete ingress default-backend-ingress
kubectl delete svc custom-404-svc mainapp-svc
kubectl delete deployment custom-404 mainapp
```
