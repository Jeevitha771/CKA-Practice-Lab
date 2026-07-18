# Q33–Q35. Advanced etcd Backup Operations

## Q33 Task
Create automated etcd backup script: snapshot every 6 hours, save to `/backup/etcd/` with timestamp, retain 7 days, log to `/var/log/etcd-backup.log`, alert on failure.

## Q34 Task
Disaster recovery test: take snapshot, create test deployment, restore from snapshot, verify test deployment is gone, document recovery time.

## Q35 Task
Verify etcd backup at `/backup/etcd-snapshot.db` without restoring: check status, display metadata, list key count.

---

## Q35: Verify a Backup Without Restoring

```bash
# Check snapshot status (metadata)
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table
```
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 4e7a8b12 |    15234 |       1021 |     4.1 MB |
+----------+----------+------------+------------+
```

```bash
# Confirm file integrity
ls -lh /backup/etcd-snapshot.db
# -rw------- 1 root root 4.1M Jan 16 10:00 /backup/etcd-snapshot.db

# REVISION = number of writes since cluster start
# TOTAL KEYS = all keys stored in etcd (including Kubernetes objects)
```

---

## Q33: Automated Backup Script

```bash
cat <<'EOF' > /usr/local/bin/etcd-backup.sh
#!/bin/bash

BACKUP_DIR="/backup/etcd"
LOG_FILE="/var/log/etcd-backup.log"
RETENTION_DAYS=7
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/etcd-backup-${TIMESTAMP}.db"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"; }

mkdir -p "$BACKUP_DIR"

log "Starting etcd backup to $BACKUP_FILE"

ETCDCTL_API=3 etcdctl snapshot save "$BACKUP_FILE" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

if [ $? -eq 0 ]; then
  log "SUCCESS: Backup saved to $BACKUP_FILE"
  # Verify the backup
  ETCDCTL_API=3 etcdctl snapshot status "$BACKUP_FILE" >> "$LOG_FILE" 2>&1
else
  log "FAILURE: etcd backup failed!"
  # Alert: send email or call alerting endpoint
  echo "etcd backup failed at $(date)" | mail -s "ALERT: etcd backup failure" admin@example.com 2>/dev/null || true
  exit 1
fi

# Retention: delete backups older than 7 days
log "Removing backups older than ${RETENTION_DAYS} days..."
find "$BACKUP_DIR" -name "etcd-backup-*.db" -mtime +${RETENTION_DAYS} -delete
log "Cleanup complete. Current backups:"
ls -lh "$BACKUP_DIR" | tee -a "$LOG_FILE"
EOF

chmod +x /usr/local/bin/etcd-backup.sh
```

### Schedule with cron (every 6 hours)

```bash
crontab -e
# Add:
0 */6 * * * /usr/local/bin/etcd-backup.sh
```

---

## Q34: Disaster Recovery Test

```bash
# Step 1: Take a snapshot (the "clean" state)
ETCDCTL_API=3 etcdctl snapshot save /opt/dr-test-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

START_TIME=$(date +%s)
echo "Snapshot taken at: $(date)"

# Step 2: Create a test deployment (this should NOT exist after restore)
kubectl create deployment dr-test --image=nginx --replicas=2
kubectl get deployment dr-test
# dr-test   2/2   Running

# Step 3: Restore from the pre-deployment snapshot (see Q19 for full restore steps)
# [restore etcd, update manifest, restart cluster]

# Step 4: Verify the test deployment is gone
kubectl get deployment dr-test
# Error from server (NotFound): deployments.apps "dr-test" not found ✅

END_TIME=$(date +%s)
echo "Recovery time: $((END_TIME - START_TIME)) seconds"
# Document: Recovery time: ~120 seconds
```
