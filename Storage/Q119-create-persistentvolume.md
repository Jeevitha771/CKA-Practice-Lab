# Q119. Create PersistentVolume `pv-data`

## Task
Create PersistentVolume `pv-data`:
- Capacity: 5Gi
- Access mode: ReadWriteOnce
- Host path: /mnt/data
- Storage class: manual
- Reclaim policy: Retain

## Solution YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
```

```bash
kubectl apply -f pv.yaml

# Verify
kubectl get pv
# STATUS should be: Available
```

## Expected Output

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS
pv-data   5Gi        RWO            Retain           Available           manual
```
