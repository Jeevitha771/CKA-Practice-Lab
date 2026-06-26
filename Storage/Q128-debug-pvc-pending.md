# Q128. Debug PVC Stuck in Pending State

## Task
PVC stuck in `Pending`. Debug: check available PVs, storage class, access modes, capacity.

## Debugging Steps

```bash
# Step 1: Check PVC status
kubectl get pvc
# NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# data-claim   Pending                                      fast-storage

# Step 2: Describe PVC for detailed error
kubectl describe pvc data-claim
# Look at the Events section for clues
```

## Common Errors and Fixes

### Error 1: StorageClass Not Found

```
Events:
  Warning  ProvisioningFailed  storageclass.storage.k8s.io "fast-storage" not found
```

**Fix — Create the StorageClass:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f sc.yaml
kubectl get pvc
```

### Error 2: No Matching PV (wrong labels, access mode, or capacity)

**Fix — Create a matching PV:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ssd
  labels:
    type: ssd
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  hostPath:
    path: /tmp/data
```

```bash
kubectl apply -f pv.yaml
kubectl get pvc
# NAME         STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS
# data-claim   Bound    pv-ssd    2Gi        RWO            fast-storage
```

## Debug Checklist

| Check | Command |
|---|---|
| PVC status | `kubectl get pvc` |
| Detailed error | `kubectl describe pvc <name>` |
| Available PVs | `kubectl get pv` |
| StorageClass exists | `kubectl get sc` |
| PV labels match PVC selector | `kubectl get pv --show-labels` |
| Access mode matches | `kubectl describe pv <name>` |
| PV capacity >= PVC request | `kubectl describe pv <name>` |
| StorageClass name matches | `kubectl get pvc -o yaml \| grep storageClassName` |
