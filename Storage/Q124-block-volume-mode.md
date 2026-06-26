# Q124. Create PV with Block Volume Mode — Demonstrate Usage in a Pod

## Task
Create PV with block volume mode. Demonstrate usage in a pod.

## Setup — Simulate Block Device (Lab Environment)

```bash
sudo dd if=/dev/zero of=/tmp/block-pv.img bs=1M count=100
sudo losetup -fP /tmp/block-pv.img
losetup -a
lsblk
```

## Step 1: Create PV with volumeMode: Block

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: raw-block-pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Block            # Raw block — no filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /dev/sdb             # Raw device path on host node
```

```bash
kubectl apply -f block-pv.yaml
```

## Step 2: Create PVC (must also specify Block mode)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block            # Must match the PV
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f block-pvc.yaml
kubectl get pvc raw-block-pvc
# STATUS should be: Bound
```

## Step 3: Use in a Pod — volumeDevices (NOT volumeMounts)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: block-disk-pod
spec:
  containers:
  - name: ubuntu-container
    image: ubuntu
    command: ["sleep", "3600"]
    volumeDevices:              # NOT volumeMounts — raw block uses volumeDevices
    - name: my-block-storage
      devicePath: /dev/xvda    # Raw device path inside container
  volumes:
  - name: my-block-storage
    persistentVolumeClaim:
      claimName: raw-block-pvc
```

```bash
kubectl apply -f block-pod.yaml

# Verify inside pod
kubectl exec -it block-disk-pod -- lsblk
kubectl exec -it block-disk-pod -- ls -l /dev/xvda

# Write raw data
kubectl exec -it block-disk-pod -- sh -c \
  'echo "hello kubernetes" > /tmp/msg && dd if=/tmp/msg of=/dev/xvda'
```

## Key Difference: Block vs Filesystem

| Feature | Filesystem (default) | Block |
|---|---|---|
| `volumeMode` | `Filesystem` | `Block` |
| Mount in pod | `volumeMounts` with `mountPath` | `volumeDevices` with `devicePath` |
| Use case | Files, logs, configs | Databases needing raw I/O performance |

## Notes
- Block mode gives the application direct access to the raw block device
- No filesystem layer → lower latency, used by high-performance databases
- `volumeDevices` is always used with block volumes; `volumeMounts` is for filesystem volumes
