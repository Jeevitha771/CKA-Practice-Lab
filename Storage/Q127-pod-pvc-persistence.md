# Q127. Create Pod Using PVC `data-claim` — Verify Data Persistence

## Task
Create pod using PVC `data-claim` mounted at `/data`.
Verify persistence: write file, delete pod, recreate pod, verify file still exists.

## Step 1: Setup PV and PVC

```yaml
# pv.yaml
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
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

```yaml
# pvc.yaml
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
      storage: 1Gi
  selector:
    matchLabels:
      type: ssd
```

## Step 2: Create Pod

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: data-volume
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: data-claim
```

```bash
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
kubectl get pod pvc-pod
```

## Step 3: Write Data Inside Pod

```bash
kubectl exec -it pvc-pod -- sh
echo "hello cka" > /data/test.txt
cat /data/test.txt
exit
```

## Step 4: Delete Pod

```bash
kubectl delete pod pvc-pod
```

## Step 5: Recreate Pod

```bash
kubectl apply -f pod.yaml
kubectl get pod pvc-pod
```

## Step 6: Verify Data Persisted

```bash
kubectl exec -it pvc-pod -- sh
cat /data/test.txt
# Output: hello cka  ✅
```

## Notes
- The data persists because it lives on the PV, not in the pod
- Deleting a Pod does NOT delete the PVC or PV
- Data is lost only if the PVC is deleted (and PV reclaim policy is Delete)
