# Q132. Configure Deployment with Multiple Volumes

## Task
Configure deployment with multiple volumes:
- ConfigMap at `/config`
- Secret at `/secrets`
- PVC at `/data`
- EmptyDir at `/tmp`

## Setup Resources

```bash
# Create ConfigMap
kubectl create configmap app-config --from-literal=app.properties="env=prod"

# Create Secret
kubectl create secret generic app-secret \
  --from-literal=username=admin \
  --from-literal=password=welcome123
```

### PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc app-pvc
```

## Deployment YAML with All 4 Volume Types

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-volume-deployment
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app-container
        image: nginx:alpine
        volumeMounts:
        - name: config-volume
          mountPath: /config
          readOnly: true          # Best practice for configs

        - name: secret-volume
          mountPath: /secrets
          readOnly: true          # Best practice for secrets

        - name: data-volume
          mountPath: /data

        - name: tmp-volume
          mountPath: /tmp

      volumes:
      - name: config-volume
        configMap:
          name: app-config        # Must match existing ConfigMap

      - name: secret-volume
        secret:
          secretName: app-secret  # Must match existing Secret

      - name: data-volume
        persistentVolumeClaim:
          claimName: app-pvc      # Must match existing PVC

      - name: tmp-volume
        emptyDir: {}              # Temporary, pod-scoped, wiped on pod restart
```

```bash
kubectl apply -f deployment.yaml

# Verify mounts inside pod
kubectl exec -it <pod-name> -- ls /config
kubectl exec -it <pod-name> -- ls /secrets
kubectl exec -it <pod-name> -- ls /data
kubectl exec -it <pod-name> -- ls /tmp
```

## Volume Type Summary

| Volume | Mount Path | Lifecycle | Use Case |
|---|---|---|---|
| ConfigMap | `/config` | ConfigMap lifetime | App config files |
| Secret | `/secrets` | Secret lifetime | Credentials, TLS |
| PVC | `/data` | PV lifetime | Persistent data |
| EmptyDir | `/tmp` | Pod lifetime | Scratch space, caching |
