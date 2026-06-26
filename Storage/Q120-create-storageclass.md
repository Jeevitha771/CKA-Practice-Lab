# Q120. Create StorageClass `fast-storage`

## Task
Create StorageClass `fast-storage`:
- Provisioner: `kubernetes.io/aws-ebs`
- Parameters: type=gp3, iopsPerGB=50
- Volume binding mode: WaitForFirstConsumer
- Reclaim policy: Delete

## Solution YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "50"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

```bash
kubectl apply -f storageclass.yaml

# Verify
kubectl get sc fast-storage
kubectl describe sc fast-storage
```

## Notes
- `WaitForFirstConsumer` → PV is not provisioned until a Pod using the PVC is scheduled
- `Immediate` → PV is provisioned as soon as PVC is created
- `reclaimPolicy: Delete` → PV and underlying storage are deleted when PVC is deleted
- `reclaimPolicy: Retain` → PV stays after PVC deletion (requires manual cleanup)
