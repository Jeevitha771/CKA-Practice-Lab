# Q129. Expand PVC from 5Gi to 10Gi

## Task
Expand PVC from 5Gi to 10Gi. Verify expansion and pod can use additional space.

## Prerequisites — StorageClass Must Allow Expansion

```bash
kubectl get sc fast-storage -o yaml
# Look for: allowVolumeExpansion: true

# If NOT present, patch the StorageClass
kubectl patch sc fast-storage -p '{"allowVolumeExpansion": true}'
```

### StorageClass YAML with expansion enabled

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/no-provisioner
allowVolumeExpansion: true
```

## PV Must Also Be Large Enough

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-ssd
  labels:
    type: ssd
spec:
  capacity:
    storage: 10Gi              # Must be >= new PVC request
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

## Expand the PVC

```bash
kubectl edit pvc data-claim
# Change:
#   spec:
#     resources:
#       requests:
#         storage: 10Gi

# Verify
kubectl get pvc data-claim
# CAPACITY should now show: 10Gi
kubectl describe pvc data-claim
```

## Verify Inside Pod

```bash
kubectl exec -it <pod-name> -- df -h /data
# Check mounted path size
```

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `only dynamically provisioned pvc can be resized` | StorageClass uses `no-provisioner` (static) | Recreate PVC with larger size |
| Pod still shows old size | Pod needs restart | Delete and recreate pod |
| `allowVolumeExpansion: false` | StorageClass doesn't allow expansion | Update StorageClass or recreate PVC |

```bash
# Verify PVC request after edit
kubectl get pvc data-claim -o jsonpath='{.spec.resources.requests.storage}'
```

## Notes
- Volume expansion is only supported with **dynamically provisioned** PVCs in most cases
- For `hostPath` / static provisioning: delete and recreate PVC with larger size
- Always delete and recreate pod after expansion if size doesn't reflect inside container
