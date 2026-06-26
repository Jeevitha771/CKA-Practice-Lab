# Q79. Use Kustomize to Manage Dev / Staging / Prod Environment Configurations

## Task
Use Kustomize to manage multiple environment configurations (dev, staging, prod) for the same application.

## Directory Structure

```
kustomize-demo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── patch-replicas.yaml
    │   └── kustomization.yaml
    └── prod/
        ├── patch-replicas.yaml
        └── kustomization.yaml
```

## Base Files

### base/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
```

### base/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### base/kustomization.yaml

```yaml
resources:
- deployment.yaml
- service.yaml
```

## Dev Overlay

### overlays/dev/patch-replicas.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
```

### overlays/dev/kustomization.yaml

```yaml
resources:
- ../../base

namePrefix: dev-           # Renames all objects: dev-web-app, dev-web-service

patchesStrategicMerge:
- patch-replicas.yaml
```

## Prod Overlay

### overlays/prod/patch-replicas.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 5
```

### overlays/prod/kustomization.yaml

```yaml
resources:
- ../../base

namePrefix: prod-          # Renames all objects: prod-web-app, prod-web-service

patchesStrategicMerge:
- patch-replicas.yaml
```

## Commands

```bash
# Preview what Dev would look like (no cluster changes)
kubectl kustomize overlays/dev
# Output: Deployment named dev-web-app with replicas=2

# Preview Prod
kubectl kustomize overlays/prod
# Output: Deployment named prod-web-app with replicas=5

# Apply Dev to cluster (uses -k instead of -f)
kubectl apply -k overlays/dev
kubectl get deployments

# Clean up Dev
kubectl delete -k overlays/dev

# Apply Prod
kubectl apply -k overlays/prod
```

## Key Concepts

| Concept | Description |
|---|---|
| **Base** | Core YAML written once — no environment-specific data |
| **Overlay** | Only the differences for each environment (tiny patches) |
| **namePrefix** | Adds a prefix to all object names per environment |
| **patchesStrategicMerge** | Deep-merges patch YAML into base YAML |
| `-k` flag | Tells kubectl to use Kustomize mode instead of plain YAML |

## Notes
- Kustomize is built into `kubectl` — no separate install needed
- Avoids copy-pasting full YAML files across environments
- If a Service port changes, edit `base/service.yaml` once → affects all envs automatically
- `kubectl kustomize` (print only) vs `kubectl apply -k` (actually apply)
