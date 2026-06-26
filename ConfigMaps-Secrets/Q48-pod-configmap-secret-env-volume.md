# Q48. Pod Using ConfigMap Key as Env Var, Secret as Env Vars, ConfigMap as Volume

## Task
Create pod using:
- ConfigMap `app-config` key `database_url` as env `DB_URL`
- Secret `db-credentials` keys as env vars (all keys)
- ConfigMap mounted at `/config`

## Setup

```bash
# Create ConfigMap
echo "database_url=postgres://db.example.com:5432/mydb" > /tmp/app.properties
kubectl -n production create configmap app-config --from-env-file=/tmp/app.properties

# Create Secret
kubectl -n production create secret generic db-credentials \
  --from-literal=username=myuser \
  --from-literal=password=mypassword
```

## Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
  namespace: production
spec:
  containers:
  - name: app-container
    image: nginx:1.21
    env:
    - name: DB_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url       # Specific key from ConfigMap
    envFrom:
    - secretRef:
        name: db-credentials      # All keys from Secret as env vars
    volumeMounts:
    - name: app-config-volume
      mountPath: /config
      readOnly: true
  volumes:
  - name: app-config-volume
    configMap:
      name: app-config
```

```bash
kubectl apply -f pod.yaml
kubectl exec -it config-pod -n production -- printenv DB_URL
kubectl exec -it config-pod -n production -- printenv username
kubectl exec -it config-pod -n production -- ls /config
```
