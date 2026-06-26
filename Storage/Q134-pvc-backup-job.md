# Q134. Backup Strategy for PVCs — Create Job to Back Up Data

## Task
Implement backup strategy for PVCs.
Create a Job that backs up data from a source PVC to a backup PVC (external location).

## Step 1: Create Source PVC (contains data to back up)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: source-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: fast-storage
```

## Step 2: Create Backup Destination PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: fast-storage
```

## Step 3: PV (if using static provisioning)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: ssd
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  hostPath:
    path: /tmp/data
```

```bash
kubectl apply -f pv.yaml
kubectl apply -f source-pvc.yaml
kubectl apply -f backup-pvc.yaml
```

## Step 4: Backup Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pvc-backup-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: backup
        image: busybox:1.36
        command:
        - sh
        - -c
        - cp -r /source/* /backup/
        volumeMounts:
        - name: source
          mountPath: /source
        - name: backup
          mountPath: /backup
      volumes:
      - name: source
        persistentVolumeClaim:
          claimName: source-data
      - name: backup
        persistentVolumeClaim:
          claimName: backup-data
```

```bash
kubectl apply -f backup-job.yaml

# Monitor job
kubectl get job pvc-backup-job
kubectl get pods -l job-name=pvc-backup-job

# Check logs
kubectl logs -l job-name=pvc-backup-job

# Verify backup completed
kubectl describe job pvc-backup-job
```

## Notes
- `restartPolicy: Never` → Job Pod does not restart on failure (use `OnFailure` for retries)
- For scheduled backups use a **CronJob** instead of a Job
- In production, copy to remote object storage (S3, GCS) using appropriate tools (aws cli, gsutil)
- Both PVCs must support simultaneous access or the source pod must be stopped during backup
