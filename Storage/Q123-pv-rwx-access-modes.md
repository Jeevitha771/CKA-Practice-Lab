# Q123. Create PV with ReadWriteMany Access Mode — Access Mode Comparison

## Task
Create PV with access mode ReadWriteMany, volume mode Filesystem, reclaim policy Recycle.
Explain when each access mode should be used.

## Solution YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rwx
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs-storage
  nfs:
    path: /exports/data
    server: 192.168.1.100
```

```bash
kubectl apply -f pv.yaml
kubectl get pv pv-rwx
```

## Access Mode Comparison

| Mode | Short | Nodes | Read | Write | Use Case |
|---|---|---|---|---|---|
| ReadWriteOnce | RWO | 1 | ✅ | ✅ | Databases, single-pod apps |
| ReadOnlyMany | ROX | Many | ✅ | ❌ | Shared static content, config |
| ReadWriteMany | RWX | Many | ✅ | ✅ | Shared storage (NFS), logs |

## Backend Provisioner Support

| Provisioner | Supported Access Modes |
|---|---|
| `hostPath` | RWO only (local node, practice only) |
| `nfs` | RWO, ROX, RWX |
| `awsElasticBlockStore` | RWO only |
| `gcePersistentDisk` | RWO, ROX |
| `azureDisk` | RWO only |

## Reclaim Policy Options

| Policy | Behaviour After PVC Deletion |
|---|---|
| `Retain` | PV stays, data preserved, manual cleanup needed |
| `Delete` | PV and storage deleted automatically |
| `Recycle` | Data scrubbed (`rm -rf`), PV made Available again (deprecated) |

## Notes
- `Recycle` is deprecated — prefer `Delete` or `Retain` instead
- `ReadWriteMany` requires a backend that supports it (NFS, CephFS, etc.)
- `hostPath` volumes only work on a single node and are not suitable for production
