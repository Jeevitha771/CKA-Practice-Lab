# Q76. Generate YAML (Don't Create) — Deployment, Service, ConfigMap

## Task
Generate YAML (don't create) for:
- Deployment `api`: 3 replicas, image `api:v1`
- Service exposing port 8080
- ConfigMap from directory `/opt/config`

## Setup Mock Config Directory

```bash
mkdir -p /opt/config
echo "environment=production" > /opt/config/app.properties
echo "database_host=mysql-primary" >> /opt/config/app.properties
```

## Generate Deployment YAML

```bash
kubectl create deployment api --image=api:v1 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

cat deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: api:v1
```

## Generate Service YAML

```bash
kubectl create service clusterip api --tcp=8080:8080 \
  --dry-run=client -o yaml > service.yaml

cat service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: api
  type: ClusterIP
```

## Generate ConfigMap from Directory

```bash
kubectl create configmap api-config --from-file=/opt/config \
  --dry-run=client -o yaml > configmap.yaml

cat configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  app.properties: |
    environment=production
    database_host=mysql-primary
```

## Notes
- `--dry-run=client -o yaml` generates valid YAML without touching the cluster
- Perfect for creating templates to edit before applying
- Use `>` to save to file, then `kubectl apply -f <file>` when ready
