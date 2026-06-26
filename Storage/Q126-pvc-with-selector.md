# Q126. Create PVC `data-claim` with Selector

## Task
Create PVC `data-claim`:
- Requests 2Gi storage
- Access mode: ReadWriteOnce
- Storage class: fast-storage
- Selector matches label `type=ssd`

## PV (must have matching label)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ssd
  labels:
    type: ssd                  # Must match PVC selector
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

## PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-claim
spec:
  storageClassName: fast-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      type: ssd                # Matches PV label type=ssd
```

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml

# Verify binding
kubectl get pvc data-claim
# STATUS should be: Bound
kubectl get pv pv-ssd
# STATUS should be: Bound
```

## Selector vs volumeName

| Method | How It Works |
|---|---|
| `selector.matchLabels` | Binds to any PV matching the labels (flexible) |
| `volumeName` | Binds to an exact PV by name (strict) |

### Direct PV binding by name

```yaml
spec:
  volumeName: pv-ssd          # Exact PV binding
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

## Notes
- `selector` in PVC must match `labels` on the PV
- If selector has no matching PV → PVC stays in `Pending`
- One PVC binds to one PV (1:1 relationship)
- Multiple Pods can use the same PVC (subject to access mode)
