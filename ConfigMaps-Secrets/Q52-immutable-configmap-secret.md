# Q52. Create Immutable ConfigMap and Secret — Observe Modification Behavior

## Task
Create an immutable ConfigMap and Secret. Attempt to modify them and observe the behavior.

## Solution

### Immutable ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  mode: production
```

### Immutable Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
immutable: true
type: Opaque
stringData:
  password: mysecret
```

```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

# Verify
kubectl get configmap app-config -o yaml
kubectl get secret app-secret -o yaml
```

## Attempt to Modify (will be rejected)

```bash
kubectl edit configmap app-config
# Error: configmap "app-config" is immutable and cannot be updated

kubectl edit secret app-secret
# Error: secret "app-secret" is immutable and cannot be updated
```

## Fix — Delete and Recreate

```bash
# For ConfigMap
kubectl delete configmap app-config
kubectl create configmap app-config --from-literal=mode=dev

# For Secret
kubectl delete secret app-secret
kubectl create secret generic app-secret --from-literal=password=newsecret

# If using replace --force
kubectl replace --force -f /tmp/kubectl-edit-45121218.yaml

# Restart deployment to pick up new config
kubectl rollout restart deployment web-app
```

## Notes
- `immutable: true` prevents any edits — the only option is delete + recreate
- Immutable ConfigMaps/Secrets improve performance at scale (no watch overhead)
- Always restart dependent deployments after recreating
