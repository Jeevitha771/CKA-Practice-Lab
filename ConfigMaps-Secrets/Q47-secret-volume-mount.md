# Q47. Create Secret `db-credentials` and Mount as Read-Only Volume

## Task
Create Secret `db-credentials` with username/password (base64 encoded).
Mount as volume at `/etc/db-credentials/` with read-only permissions.

## Solution

```bash
# Step 1: Create the secret (kubectl encodes to base64 automatically)
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret

# Step 2: Verify (values will be base64 encoded)
kubectl get secret db-credentials -o yaml
```

## Pod YAML — Mount Secret as Read-Only Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/db-credentials/
      readOnly: true              # Ensure it is read-only
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials  # Matches the secret created above
```

```bash
kubectl apply -f pod.yaml

# Verify files are mounted
kubectl exec secret-pod -- ls -la /etc/db-credentials/

# Read a secret value
kubectl exec secret-pod -- cat /etc/db-credentials/username
```

## Notes
- Each key in the Secret becomes a file under the mountPath
- `readOnly: true` prevents the container from writing to the mount
- Values are base64-encoded at rest; decoded automatically when mounted
