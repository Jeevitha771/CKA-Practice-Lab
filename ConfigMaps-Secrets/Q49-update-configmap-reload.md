# Q49. Update ConfigMap and Reload Pods Without Manual Deletion

## Task
Update ConfigMap used by a running deployment. Force pods to reload new configuration without manual deletion.

## Solution

```bash
# Option 1: Edit ConfigMap in place
kubectl edit configmap app-config

# Option 2: Apply updated YAML
kubectl apply -f configmap.yaml

# Force pods to restart and pick up new config
kubectl rollout restart deployment web-app
kubectl rollout status deployment web-app

# Verify new env var inside pod
kubectl exec -it <pod-name> -- printenv
```

## Tricky Exam Variations

| Scenario | Solution |
|---|---|
| "Ensure application picks new config automatically" | `kubectl rollout restart deployment web-app` |
| "ConfigMap values should reflect without restarting pods" | Use **volume mount** (files update automatically) |
| "Ensure pods restart automatically when ConfigMap changes" | Patch annotation to trigger rollout |
| "Load ConfigMap + Secret into pod" | `env` (configMapKeyRef) + `envFrom` (secretRef) |
| "Use one key from ConfigMap and all keys from Secret" | `env` → specific key; `envFrom` → all keys |
| "Mounted ConfigMap Not Updating Immediately" | `kubectl rollout restart deployment web-app` |
| "Immutable ConfigMap cannot be edited" | Delete + recreate + rollout restart |
| "ConfigMap Not Found Error" | `kubectl create configmap app-config --from-literal=key=value` |
| "Update config without downtime" | `kubectl rollout restart deployment web-app` |

## Patch Annotation Trick (auto-restart on config change)

```bash
kubectl patch deployment web-app \
  -p '{"spec":{"template":{"metadata":{"annotations":{"configmap-version":"1"}}}}}'
```

## Notes
- **envFrom** (env vars): pods must restart to pick up ConfigMap changes
- **volumeMount**: files update in-place ~1 minute after ConfigMap update (no restart needed)
