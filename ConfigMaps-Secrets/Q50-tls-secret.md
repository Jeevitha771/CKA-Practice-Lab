# Q50. Create TLS Secret from Certificate Files

## Task
Create TLS secret from files:
- `tls.crt` from `/opt/certs/server.crt`
- `tls.key` from `/opt/certs/server.key`
- type: `kubernetes.io/tls`

## Solution

```bash
# Step 1: Create certs directory
mkdir -p /opt/certs

# Step 2: Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/certs/server.key \
  -out /opt/certs/server.crt \
  -subj "/CN=myapp"

# Step 3: Verify files exist
ls /opt/certs/

# Step 4: Create TLS secret
kubectl create secret tls my-tls-secret \
  --cert=/opt/certs/server.crt \
  --key=/opt/certs/server.key

# Step 5: Verify
kubectl describe secret my-tls-secret
kubectl get secret my-tls-secret -o yaml
```

## Secret YAML Structure

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded cert>
  tls.key: <base64 encoded key>
```

## Notes
- `kubectl create secret tls` automatically sets `type: kubernetes.io/tls`
- Keys are always named `tls.crt` and `tls.key` (fixed by the type)
- Commonly used with Ingress resources for HTTPS termination
