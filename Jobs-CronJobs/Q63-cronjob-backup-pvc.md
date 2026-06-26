# Q63. Create CronJob: Daily 2 AM Backup, 3 Successful / 1 Failed History, Forbid Concurrency

## Task
Create CronJob that:
- Runs daily at **2 AM**
- Backs up PVC data
- Keeps **3 successful** / **1 failed** job history
- `activeDeadlineSeconds`: 3600
- `concurrencyPolicy`: Forbid

## Solution

```yaml
# PV for source data
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-backup-source
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/kubernetes/backup-data"

---
# PVC for source data
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: source-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

---
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pvc-backup-cronjob
spec:
  schedule: "0 2 * * *"           # Run at 2:00 AM every day
  concurrencyPolicy: Forbid        # Skip new run if previous still running
  successfulJobsHistoryLimit: 3    # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1        # Keep last 1 failed job

  jobTemplate:
    spec:
      activeDeadlineSeconds: 3600  # Job times out after 1 hour
      template:
        spec:
          containers:
          - name: backup-container
            image: busybox
            command: ["/bin/sh", "-c", "echo 'Backing up PVC data...'; sleep 10"]
            volumeMounts:
            - name: data-volume
              mountPath: /data
          volumes:
          - name: data-volume
            persistentVolumeClaim:
              claimName: source-data-pvc
          restartPolicy: OnFailure
```

```bash
kubectl apply -f cronjob.yaml

# Verify CronJob
kubectl get cronjob pvc-backup-cronjob

# Manually trigger a run to test
kubectl create job --from=cronjob/pvc-backup-cronjob manual-backup-test

# Watch job pods
kubectl get pods -w
kubectl get jobs
```

## Cron Schedule Format

```
┌─────── Minute   (0-59)
│ ┌───── Hour     (0-23)
│ │ ┌─── Day      (1-31)
│ │ │ ┌─ Month    (1-12)
│ │ │ │ ┌ Weekday (0-7, 0=Sun)
│ │ │ │ │
0 2 * * *   →  2:00 AM every day
```

## Concurrency Policy Options

| Policy | Behaviour |
|---|---|
| `Allow` (default) | Start new job even if previous is still running |
| `Forbid` | Skip new run if previous is still running |
| `Replace` | Kill running job and start new one |
