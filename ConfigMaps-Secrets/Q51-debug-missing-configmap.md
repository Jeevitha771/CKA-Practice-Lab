# Q51. Debug Pod Failing Due to Missing ConfigMap

## Task
Pod cannot start due to missing ConfigMap. Create the missing ConfigMap with appropriate data, verify pod starts.

## Debugging Steps

```bash
# Step 1: Check pod status
kubectl get pods

# Step 2: Describe to find the error
kubectl describe pod <pod-name>
# Look for: Error: configmap "app-config" not found
# Or:       Error: couldn't find key "wrongkey" in ConfigMap

# Step 3: Identify the required ConfigMap name and key from pod spec
kubectl get pod app-pod -o yaml | grep -A5 configMapKeyRef
```

## Common Error in Pod Spec

```yaml
env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config    # ConfigMap must exist
        key: wrongkey       # Key must exist in the ConfigMap
```

## Fix

```bash
# Create the missing ConfigMap with correct key
kubectl create configmap app-config --from-literal=mode=production

# Verify ConfigMap exists
kubectl get configmap
kubectl describe configmap app-config
```

## Pod YAML (corrected)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-config-test
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: mode          # Must match key in ConfigMap
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl exec -it app-config-test -- printenv APP_MODE
# Output: production
```

## Notes
- If pod is already running with wrong key, delete and recreate it
- If ConfigMap is immutable, you must delete and recreate it
  ```bash
  kubectl replace --force -f /tmp/kubectl-edit-*.yaml
  ```
