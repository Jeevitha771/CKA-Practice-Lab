# Q131. Create StatefulSet `database` with VolumeClaimTemplates

## Task
Create StatefulSet `database`:
- 3 replicas
- VolumeClaimTemplate requesting 10Gi storage
- Each pod gets its own PVC
- Mount path: `/var/lib/mysql`

## Full YAML

```yaml
# 1. Headless Service (required by StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: database
  labels:
    app: database
spec:
  ports:
  - port: 3306
  clusterIP: None              # Headless — enables stable DNS per pod
  selector:
    app: database

---
# 2. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  selector:
    matchLabels:
      app: database
  serviceName: database        # Must match headless service name
  replicas: 3
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "secretpassword"
        volumeMounts:
        - name: database-storage
          mountPath: /var/lib/mysql    # Requirement: mount path

  # Each pod gets its own PVC from this template
  volumeClaimTemplates:
  - metadata:
      name: database-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi              # Requirement: 10Gi per pod
```

```bash
kubectl apply -f statefulset.yaml

# Verify pods come up sequentially
kubectl get pods -w -l app=database

# Verify each pod has its own PVC
kubectl get pvc
# Expected:
#   database-storage-database-0
#   database-storage-database-1
#   database-storage-database-2
```

## PVC Naming Convention

```
<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>
database-storage-database-0
database-storage-database-1
database-storage-database-2
```

## Notes
- StatefulSet pods start in order: database-0 → database-1 → database-2
- Each pod has a stable DNS: `database-0.database.default.svc.cluster.local`
- PVCs are NOT deleted when StatefulSet is scaled down (data preserved)
- Headless service (`clusterIP: None`) is required for stable network identities
