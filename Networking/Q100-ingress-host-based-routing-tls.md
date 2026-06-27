# Q100. Ingress with Host-Based Routing — Each Host with Own TLS

## Task
Create an Ingress with host-based routing:
- `api.example.com` → `api-service:8080`
- `web.example.com` → `web-service:80`
- `admin.example.com` → `admin-service:8080`

Each host has its own TLS secret.

---

## Phase 1: Deploy Backend Services

```bash
kubectl create deployment api-app   --image=nginx
kubectl create deployment web-app   --image=nginx
kubectl create deployment admin-app --image=nginx

kubectl expose deployment api-app   --name=api-service   --port=8080 --target-port=80
kubectl expose deployment web-app   --name=web-service   --port=80
kubectl expose deployment admin-app --name=admin-service --port=8080 --target-port=80
```

---

## Phase 2: Create a TLS Secret Per Host

```bash
# api.example.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/api.key -out /tmp/api.crt \
  -subj "/CN=api.example.com"
kubectl create secret tls api-tls --cert=/tmp/api.crt --key=/tmp/api.key

# web.example.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/web.key -out /tmp/web.crt \
  -subj "/CN=web.example.com"
kubectl create secret tls web-tls --cert=/tmp/web.crt --key=/tmp/web.key

# admin.example.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/admin.key -out /tmp/admin.crt \
  -subj "/CN=admin.example.com"
kubectl create secret tls admin-tls --cert=/tmp/admin.crt --key=/tmp/admin.key
```

Verify all 3 secrets:
```bash
kubectl get secret api-tls web-tls admin-tls
```

---

## Phase 3: Create the Host-Based Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls           # cert for api subdomain
  - hosts:
    - web.example.com
    secretName: web-tls           # cert for web subdomain
  - hosts:
    - admin.example.com
    secretName: admin-tls         # cert for admin subdomain
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
EOF
```

**Key difference from path-based routing:**
- Each `host:` entry under `rules` handles a completely different virtual host.
- Each `tls:` entry has its own `secretName` — SNI (Server Name Indication) allows
  the Nginx controller to serve the correct cert per hostname.

---

## Phase 4: Verify

```bash
kubectl get ingress multi-host-ingress
```

```
NAME                 CLASS   HOSTS                                                      ADDRESS        PORTS     AGE
multi-host-ingress   nginx   api.example.com,web.example.com,admin.example.com          192.168.1.10   80, 443   30s
```

```bash
kubectl describe ingress multi-host-ingress
```

```
TLS:
  api-tls   terminates api.example.com
  web-tls   terminates web.example.com
  admin-tls terminates admin.example.com
Rules:
  Host                  Path   Backends
  api.example.com       /      api-service:8080
  web.example.com       /      web-service:80
  admin.example.com     /      admin-service:8080
```

---

## Phase 5: Test Each Host

```bash
HTTPS_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

curl -k -H "Host: api.example.com"   https://$NODE_IP:$HTTPS_PORT
curl -k -H "Host: web.example.com"   https://$NODE_IP:$HTTPS_PORT
curl -k -H "Host: admin.example.com" https://$NODE_IP:$HTTPS_PORT
```

---

## Host-Based vs Path-Based Routing

| Aspect | Path-Based | Host-Based |
|---|---|---|
| Differentiation | URL path (`/api`, `/web`) | Hostname (`api.`, `web.`) |
| Single host | ✅ Yes — one host, many paths | ❌ No — different hostnames |
| TLS | One cert covers all paths | One cert per hostname (via SNI) |
| Use case | Monorepo API under one domain | Separate microservice subdomains |

---

## Summary: Quick Reference

```bash
# Create secrets
kubectl create secret tls api-tls   --cert=api.crt   --key=api.key
kubectl create secret tls web-tls   --cert=web.crt   --key=web.key
kubectl create secret tls admin-tls --cert=admin.crt --key=admin.key

# Create ingress
kubectl apply -f multi-host-ingress.yaml

# Verify
kubectl get ingress multi-host-ingress
kubectl describe ingress multi-host-ingress

# Test
curl -k -H "Host: api.example.com" https://<NODE-IP>:<PORT>
```

---

## Cleanup

```bash
kubectl delete ingress multi-host-ingress
kubectl delete secret api-tls web-tls admin-tls
kubectl delete svc api-service web-service admin-service
kubectl delete deployment api-app web-app admin-app
```
