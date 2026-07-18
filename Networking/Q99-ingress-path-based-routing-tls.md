# Q99. Ingress with Path-Based Routing and TLS

## Task
Create an Ingress for host `app.example.com` with:
- `/api` ΓåÆ `api-service:8080`
- `/web` ΓåÆ `web-service:80`
- `/admin` ΓåÆ `admin-service:8080`
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
# or
To thoroughly test this Ingress, you need to verify two main things: that Kubernetes has correctly parsed your rules, and that actual network traffic is routing to the right places.

Because you are using a dummy domain (`app.example.com`) and TLS, you can't just type this into your browser immediately. Here is the step-by-step guide to testing this configuration effectively.

### Phase 1: Apply and Inspect

First, make sure the Ingress is accepted by your cluster and has been assigned an IP address by your Ingress Controller (like NGINX).

1. **Apply the file:**
```bash
kubectl apply -f ingress.yaml

```


2. **Check the status:**
```bash
kubectl get ingress app-ingress

```


*Wait until you see an IP address or hostname under the `ADDRESS` column. If it stays blank for a long time, your Ingress Controller might not be running.*
3. **Inspect the routing rules:**
```bash
kubectl describe ingress app-ingress

```


*Look at the "Rules" section in the output. Ensure it correctly lists your paths mapping to your backend services. Also, check the "Events" at the bottom for any syntax errors or warnings.*

---

### Phase 2: The "Fake Domain" Workaround

Since `app.example.com` isn't a real domain pointing to your cluster, your computer doesn't know where to send the traffic. You have two ways to bypass this for testing:

**Method A: The `curl` resolve trick (Quickest for CLI)**
You can force `curl` to send traffic to your Ingress IP while pretending the host is `app.example.com`.
*(Note: Replace `<INGRESS_IP>` with the IP you got from Phase 1).*

```bash
# Test the /api route (using -k to bypass self-signed cert warnings)
curl -kv --resolve app.example.com:443:<INGRESS_IP> https://app.example.com/api

# Test the /web route
curl -kv --resolve app.example.com:443:<INGRESS_IP> https://app.example.com/web

```

**Method B: Modify your `/etc/hosts` file (Best for Browser testing)**
If you want to test this in your web browser, you need to map the domain to your Ingress IP locally on your machine.

1. Open your hosts file as an administrator (`/etc/hosts` on Mac/Linux, `C:\Windows\System32\drivers\etc\hosts` on Windows).
2. Add this line at the bottom:
```text
<INGRESS_IP>    app.example.com

```


3. Save the file. Now you can open Chrome/Firefox and navigate to `https://app.example.com/web`.

---

### Phase 3: What to Look For During the Test

When you run your tests, verify the following behaviors:

* **HTTP 200 OK:** If you hit `/api`, do you get a successful response from `api-service`?
* **HTTP 404 Not Found:** If you hit `https://app.example.com/fake-path`, does the Ingress properly reject it?
* **Certificate Check:** Because you used the `-v` (verbose) flag in `curl`, look at the SSL handshake logs. Does it show that it is serving the certificate from your `tls-secret`?
* *(Note: If you haven't created the `tls-secret` yet, NGINX will serve a "Fake Kubernetes Ingress" default certificate).*

### Common Pitfalls to Watch Out For

* **Missing Backends:** If your `api-service` or `web-service` pods are crashing or don't exist, the Ingress will return a `502 Bad Gateway` or `503 Service Unavailable`.
* **No SSL Redirect:** Currently, your Ingress doesn't explicitly force HTTP traffic to upgrade to HTTPS. If you test `http://app.example.com`, it might just route unencrypted.

Are you using an Ingress Controller like NGINX, and do you already have your backend services and pods running, or do we need to set up dummy pods to test this against?

---
## Path Routing Logic

```
Request: https://app.example.com/api/users
         Γåô
Nginx controller matches: path=/api (Prefix)
         Γåô
Forwards to: api-service:8080/users  (rewrite strips /api prefix)
```

### `pathType` options:

| pathType | Behaviour |
|---|---|
| `Prefix` | Matches path and all sub-paths (`/api`, `/api/users`, `/api/v2/...`) |
| `Exact` | Only matches the exact path ΓÇö `/api` only, not `/api/users` |
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
