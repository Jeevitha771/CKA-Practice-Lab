# Q99. Ingress with Path-Based Routing and TLS

## Task
Create an Ingress for host `app.example.com` with:
- `/api` → `api-service:8080`
- `/web` → `web-service:80`
- `/admin` → `admin-service:8080`
- TLS using secret `tls-secret`

---

## Phase 1: Deploy Backend Services

```bash
# Create deployments
kubectl create deployment api-app   --image=nginx
kubectl create deployment web-app   --image=nginx
kubectl create deployment admin-app --image=nginx

# Expose as services
kubectl expose deployment api-app   --name=api-service   --port=8080 --target-port=80
kubectl expose deployment web-app   --name=web-service   --port=80
kubectl expose deployment admin-app --name=admin-service --port=8080 --target-port=80
```

---

## Phase 2: Create the TLS Secret

```bash
# Generate a self-signed certificate for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key \
  -out /tmp/tls.crt \
  -subj "/CN=app.example.com/O=test"

# Create the TLS secret
kubectl create secret tls tls-secret \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key
```

Verify:
```bash
kubectl get secret tls-secret
```

```
NAME         TYPE                DATA   AGE
tls-secret   kubernetes.io/tls   2      10s
```

---

## Phase 3: Create the Path-Based Ingress with TLS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret        # TLS cert for this host
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
EOF
```

**Field breakdown:**

| Field | Purpose |
|---|---|
| `spec.tls` | Binds the TLS certificate to the host |
| `secretName: tls-secret` | Which k8s Secret holds the cert/key |
| `path: /api` | URL prefix that routes to api-service |
| `pathType: Prefix` | Match any path starting with `/api` |
| `rewrite-target: /` | Strip the path prefix before forwarding to the backend |

---

## Phase 4: Verify

```bash
kubectl get ingress app-ingress
```

```
NAME          CLASS   HOSTS             ADDRESS        PORTS     AGE
app-ingress   nginx   app.example.com   192.168.1.10   80, 443   30s
```

> `PORTS: 80, 443` confirms TLS is configured.

```bash
kubectl describe ingress app-ingress
```

```
TLS:
  tls-secret terminates app.example.com
Rules:
  Host             Path   Backends
  app.example.com
                   /api   api-service:8080
                   /web   web-service:80
                   /admin admin-service:8080
```

---

## Phase 5: Test Each Path

```bash
# Get NodePort for HTTPS
HTTPS_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')

NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

# Test each path (-k skips TLS verification for self-signed cert)
curl -k -H "Host: app.example.com" https://$NODE_IP:$HTTPS_PORT/api
curl -k -H "Host: app.example.com" https://$NODE_IP:$HTTPS_PORT/web
curl -k -H "Host: app.example.com" https://$NODE_IP:$HTTPS_PORT/admin
```

Each should return `200 OK` from the corresponding nginx backend.

---

## Path Routing Logic

```
Request: https://app.example.com/api/users
         ↓
Nginx controller matches: path=/api (Prefix)
         ↓
Forwards to: api-service:8080/users  (rewrite strips /api prefix)
```

### `pathType` options:

| pathType | Behaviour |
|---|---|
| `Prefix` | Matches path and all sub-paths (`/api`, `/api/users`, `/api/v2/...`) |
| `Exact` | Only matches the exact path — `/api` only, not `/api/users` |
| `ImplementationSpecific` | Behaviour depends on the Ingress controller |

---

## Summary: Quick Reference

```bash
# Create TLS secret
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key

# Create ingress
kubectl apply -f app-ingress.yaml

# Verify
kubectl get ingress app-ingress
kubectl describe ingress app-ingress

# Test
curl -k -H "Host: app.example.com" https://<NODE-IP>:<HTTPS-PORT>/api
```

---

## Cleanup

```bash
kubectl delete ingress app-ingress
kubectl delete secret tls-secret
kubectl delete svc api-service web-service admin-service
kubectl delete deployment api-app web-app admin-app
```
