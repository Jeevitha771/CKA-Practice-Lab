# Q46. Create ConfigMap `app-config` from Multiple Sources

## Task
Create ConfigMap `app-config` in namespace `production`:
- Literal values: `database_url`, `log_level`
- From file: `/opt/config/app.properties`
- From env file: `/opt/config/app.env`
- Use in pod as environment variables

## Solution

```bash
# Step 1: Create config directory and files
mkdir -p /opt/config

# Create app.properties
echo -e "app.name=demo-app\napp.version=1.0.0" > /opt/config/app.properties

# Create app.env
echo -e "ENVIRONMENT=production\nDEBUG=false" > /opt/config/app.env

# Step 2: Create ConfigMap from all sources
kubectl create configmap app-config \
  --namespace=production \
  --from-literal=database_url='postgres://db.example.com:5432/mydb' \
  --from-literal=log_level='INFO' \
  --from-file=/opt/config/app.properties \
  --from-env-file=/opt/config/app.env

# Step 3: Verify
kubectl -n production describe configmap app-config
```

## Use in Pod as Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: production
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name: app-config
```

```bash
kubectl apply -f pod.yaml
kubectl exec -n production nginx-pod -- env
```

## Imperative: Import env from ConfigMap into Deployment

```bash
kubectl set env --from=configmap/app-config deployment/web-app -n production
```

## Notes
- `--from-literal` → individual key=value pairs
- `--from-file` → file content stored under filename as key
- `--from-env-file` → each line `KEY=VALUE` becomes a separate key
