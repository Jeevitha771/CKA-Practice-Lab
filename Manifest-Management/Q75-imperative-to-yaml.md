# Q75. Convert Imperative Command to YAML

## Task
Convert this imperative command to YAML:
```bash
kubectl run nginx --image=nginx:1.21 --port=80 \
  --env="ENV=production" \
  --labels="app=web,tier=frontend" \
  --requests="cpu=100m,memory=128Mi" \
  --limits="cpu=200m,memory=256Mi"
```

## Equivalent YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    env:
    - name: ENV
      value: production
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

## Flag-to-YAML Mapping

| Flag | YAML Location |
|---|---|
| `kubectl run <name>` | `metadata.name` |
| `--image` | `spec.containers[].image` |
| `--port` | `spec.containers[].ports[].containerPort` |
| `--env="KEY=VALUE"` | `spec.containers[].env[]` |
| `--labels="k=v"` | `metadata.labels` |
| `--requests="cpu=...,memory=..."` | `spec.containers[].resources.requests` |
| `--limits="cpu=...,memory=..."` | `spec.containers[].resources.limits` |

## Generate YAML with --dry-run (Exam Tip)

```bash
kubectl run nginx --image=nginx:1.21 --port=80 \
  --env="ENV=production" \
  --labels="app=web,tier=frontend" \
  --requests="cpu=100m,memory=128Mi" \
  --limits="cpu=200m,memory=256Mi" \
  --dry-run=client -o yaml > generated-pod.yaml

cat generated-pod.yaml
kubectl apply -f generated-pod.yaml
kubectl get pods --show-labels
kubectl describe pod nginx
```
