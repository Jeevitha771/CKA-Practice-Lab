# Q101. Create TLS Secret from Files and Configure Ingress for HTTPS

## Task
- Create a TLS secret from existing files:
  - Certificate: `/opt/certs/tls.crt`
  - Key: `/opt/certs/tls.key`
  - Secret name: `app-tls`
- Configure an Ingress to use this secret for HTTPS

---

## Phase 1: Prepare the Certificate Files (Lab Setup)

If the cert files don't exist yet, create them:

```bash
mkdir -p /opt/certs

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/certs/tls.key \
  -out /opt/certs/tls.crt \
  -subj "/CN=app.example.com/O=myorg"
```

Verify files exist:
```bash
ls -la /opt/certs/
# tls.crt  tls.key
```

---

## Phase 2: Create the TLS Secret

```bash
kubectl create secret tls app-tls \
  --cert=/opt/certs/tls.crt \
  --key=/opt/certs/tls.key
```

> This is the exact command format the CKA exam will test.
> `--cert` = certificate file, `--key` = private key file.

### Verify the secret

```bash
kubectl get secret app-tls
```

```
NAME      TYPE                DATA   AGE
app-tls   kubernetes.io/tls   2      5s
```

```bash
kubectl describe secret app-tls
```

```
Name:         app-tls
Type:         kubernetes.io/tls
Data
====
tls.crt:  1234 bytes
tls.key:  1679 bytes
```

> The secret stores two keys: `tls.crt` and `tls.key` — exactly what the Ingress controller reads.

---

## Phase 3: Inspect the Certificate (Optional)

```bash
# Decode the certificate to verify it matches
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep -E "Subject:|Not After"
```

```
Subject: CN=app.example.com, O=myorg
Not After : Jan 01 00:00:00 2026 GMT
```

---

## Phase 4: Create an HTTPS Ingress Using the Secret

```bash
# First deploy a backend
kubectl create deployment myapp --image=nginx
kubectl expose deployment myapp --name=myapp-service --port=80

# Create HTTPS Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls            # references the TLS secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
EOF
```

---

## Phase 5: Verify

```bash
kubectl get ingress https-ingress
```

```
NAME            CLASS   HOSTS             ADDRESS        PORTS     AGE
https-ingress   nginx   app.example.com   192.168.1.10   80, 443   20s
```

> `PORTS: 80, 443` confirms HTTPS is configured.

```bash
kubectl describe ingress https-ingress | grep -A 3 TLS
```

```
TLS:
  app-tls terminates app.example.com
```

---

## Phase 6: Test HTTPS Access

```bash
HTTPS_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

# -k ignores self-signed cert warning
curl -kv -H "Host: app.example.com" https://$NODE_IP:$HTTPS_PORT 2>&1 | grep -E "SSL|HTTP|200"
```

Expected:
```
* SSL connection using TLS1.3 / ...
* Server certificate: CN=app.example.com
< HTTP/1.1 200 OK
```

---

## Summary: Key Commands

```bash
# Create TLS secret from files (exam command)
kubectl create secret tls app-tls \
  --cert=/opt/certs/tls.crt \
  --key=/opt/certs/tls.key

# Verify
kubectl get secret app-tls
kubectl describe secret app-tls

# Check cert contents
kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

---

## Cleanup

```bash
kubectl delete ingress https-ingress
kubectl delete secret app-tls
kubectl delete service myapp-service
kubectl delete deployment myapp
```
