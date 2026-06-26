# Q65. Create StatefulSet: 3 Replicas, Headless Service, 10Gi VolumeClaimTemplate, Parallel Pod Management, RollingUpdate Partition=1

## Task
Create StatefulSet for database:
- 3 replicas
- Headless Service
- VolumeClaimTemplate: 10Gi
- `podManagementPolicy: Parallel`
- `updateStrategy: RollingUpdate` with `partition: 1`

## Solution

```yaml
# 1. Headless Service
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None        # Makes this headless — direct DNS per pod
  selector:
    app: database
  ports:
  - port: 5432
    name: db-port

---
# 2. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db-statefulset
spec:
  serviceName: "db-headless"       # Must match headless service name
  replicas: 3

  podManagementPolicy: Parallel    # Start all pods simultaneously (default: OrderedReady)

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 1                 # Only update pods with ordinal >= 1 (pod-1, pod-2)
                                   # pod-0 stays on old version until partition is lowered

  selector:
    matchLabels:
      app: database

  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          value: "supersecret"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:
  - metadata:
      name: db-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

```bash
kubectl apply -f statefulset.yaml

# Watch pods come up in parallel
kubectl get pods -l app=database -w

# Verify each pod has its own PVC
kubectl get pvc
# db-data-db-statefulset-0
# db-data-db-statefulset-1
# db-data-db-statefulset-2
```

## Concept Summary

| Feature | Default | This Question |
|---|---|---|
| Pod startup order | `OrderedReady` (one by one) | `Parallel` (all at once) |
| Update scope with `partition: 1` | All pods | Only pods with ordinal ≥ 1 |
| DNS per pod | `pod-name.service-name.ns.svc.cluster.local` | Direct, stable DNS |
| PVC per pod | Shared | Dedicated (one per ordinal) |

## partition: 1 Explained

- **Before partition is applied:** Update image → pod-1 and pod-2 get new image, pod-0 stays old
- **Phased rollout:** Verify pod-1 and pod-2 are stable, then set `partition: 0` to update pod-0
- **Canary for StatefulSets**
