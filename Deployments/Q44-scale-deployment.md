# Q44. Scale Deployment Using kubectl scale and kubectl edit

## Task
Scale `web-app` to 5 replicas using `kubectl scale`. Then edit deployment to scale to 8. Verify both methods.

## Solution

### Method 1: kubectl scale

```bash
kubectl scale deployment web-app --replicas=5
kubectl get deployment web-app
kubectl describe deployment web-app
```

### Method 2: kubectl edit

```bash
kubectl edit deployment web-app
# In the editor, find and change:
#   spec:
#     replicas: 8
```

### Verify

```bash
kubectl get deployment web-app
kubectl get pods -l app=web
kubectl get pods -w
```

## Notes
- `kubectl scale` — quick, imperative, ideal for exams
- `kubectl edit` — opens live YAML in vi; change `spec.replicas`
- Both methods are equivalent
