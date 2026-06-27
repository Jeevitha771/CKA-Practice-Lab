# Q103. Configure Ingress Annotations — SSL Redirect, Timeout, Body Size, Rate Limiting

## Task
Configure an Ingress with Nginx annotations for:
- SSL redirect (HTTP → HTTPS)
- Connection timeout: `60s`
- Proxy body size: `10m`
- Rate limiting: `10 req/s`

> **SKIM note:** Annotation specifics are Nginx-controller-dependent. For the CKA exam,
> know the annotation format and where to add them. Deep memorisation is not required.

---

## The Annotated Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Force HTTP → HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Connection/proxy timeouts (in seconds)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # Max allowed request body size (e.g. for file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # Rate limiting: 10 requests per second per IP
    nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-svc
            port:
              number: 80
EOF
```

---

## Annotation Reference

| Annotation | Value | Effect |
|---|---|---|
| `ssl-redirect` | `"true"` | Redirect HTTP requests to HTTPS |
| `force-ssl-redirect` | `"true"` | Redirect even when behind a proxy |
| `proxy-connect-timeout` | `"60"` | Seconds to wait when connecting to backend |
| `proxy-send-timeout` | `"60"` | Seconds to wait sending a request to backend |
| `proxy-read-timeout` | `"60"` | Seconds to wait for backend response |
| `proxy-body-size` | `"10m"` | Max request body size (0 = unlimited) |
| `limit-rps` | `"10"` | Max requests per second per client IP |

---

## Verify

```bash
kubectl describe ingress annotated-ingress | grep -A 20 Annotations
```

```
Annotations:
  nginx.ingress.kubernetes.io/ssl-redirect:        true
  nginx.ingress.kubernetes.io/proxy-body-size:     10m
  nginx.ingress.kubernetes.io/proxy-read-timeout:  60
  nginx.ingress.kubernetes.io/limit-rps:           10
```

## Test SSL Redirect

```bash
HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

# HTTP request should get 308 redirect to HTTPS
curl -v -H "Host: app.example.com" http://$NODE_IP:$HTTP_PORT 2>&1 | grep -E "< HTTP|Location"
```

```
< HTTP/1.1 308 Permanent Redirect
< Location: https://app.example.com/
```

---

## Cleanup

```bash
kubectl delete ingress annotated-ingress
```
