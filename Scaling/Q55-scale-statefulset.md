# Q55. Scale StatefulSet `database` from 3 to 5 Replicas

## Task
Scale StatefulSet `database` from 3 to 5 replicas.
Verify each pod gets a unique PVC, naming follows the pattern, and scaling is sequential.

## Setup — StatefulSet with VolumeClaimTemplates

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  labels:
    app: database
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: database
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  selector:
    matchLabels:
      app: database
  serviceName: "database-svc"
  replicas: 3
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: database
    spec:
      terminationGracePeriodSeconds: 10
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
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: database-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

## Scale Command

```bash
# Scale from 3 to 5
kubectl scale statefulset database --replicas=5

# Watch pods come up sequentially (database-0, database-1 ... database-4)
kubectl get pods -l app=database -w

# Verify each pod has its own PVC
kubectl get pvc
# Expected naming pattern:
#   database-storage-database-0
#   database-storage-database-1
#   database-storage-database-2
#   database-storage-database-3
#   database-storage-database-4
```

## Notes
- StatefulSets scale **sequentially** (each pod must be Running before next starts)
- Each pod gets its own PVC named: `<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`
- Scaling down removes pods in reverse order (database-4 first)
- PVCs are **not deleted** when scaling down (data is preserved)

## Traffic Routing
- ClusterIP → kube-proxy → distributes to one of the pods
- Headless Service (`clusterIP: None`) → DNS returns individual pod IPs directly
