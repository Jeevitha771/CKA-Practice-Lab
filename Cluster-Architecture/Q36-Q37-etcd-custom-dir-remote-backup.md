# Q36–Q37. etcd Custom Data Directory and Remote Backup

## Q36 Task
Configure etcd to use data directory `/data/etcd` instead of default `/var/lib/etcd`. Update static pod manifest and verify.

## Q37 Task
Take an etcd snapshot and copy it to a remote backup server using scp. Automate this process.

---

## Q36: Change etcd Data Directory

### Step 1: Take a Backup First

```bash
# Always backup before making changes
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-pre-change.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Step 2: Create the New Directory

```bash
mkdir -p /data/etcd
# Ensure etcd process can write to it (owned by root or etcd user)
chown -R root:root /data/etcd
chmod 700 /data/etcd
```

### Step 3: Update etcd Static Pod Manifest

```bash
vi /etc/kubernetes/manifests/etcd.yaml
```

Change these two locations:

```yaml
# 1. In spec.containers[0].command:
# Before:
- --data-dir=/var/lib/etcd
# After:
- --data-dir=/data/etcd

# 2. In spec.volumes (hostPath):
# Before:
- hostPath:
    path: /var/lib/etcd
    type: DirectoryOrCreate
  name: etcd-data
# After:
- hostPath:
    path: /data/etcd
    type: DirectoryOrCreate
  name: etcd-data
```

### Step 4: kubelet Detects the Change and Restarts etcd

```bash
# kubelet watches /etc/kubernetes/manifests/ automatically
# Wait ~30s for etcd to restart with new data dir
watch crictl ps | grep etcd

# Verify etcd is running from new directory
ps aux | grep etcd | grep data-dir
# --data-dir=/data/etcd
```

### Step 5: Verify Cluster Functionality

```bash
kubectl get nodes
kubectl get pods -n kube-system

ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Q37: Snapshot + Remote Backup via scp

### One-Time Remote Backup

```bash
# Step 1: Take snapshot
SNAPSHOT="/opt/etcd-backup-$(date +%Y%m%d_%H%M%S).db"

ETCDCTL_API=3 etcdctl snapshot save "$SNAPSHOT" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Step 2: Copy to remote backup server
scp "$SNAPSHOT" backup-user@192.168.2.100:/backups/etcd/

echo "Backup sent: $SNAPSHOT"
```

---

### Automated Script with scp

```bash
cat <<'EOF' > /usr/local/bin/etcd-remote-backup.sh
#!/bin/bash

SNAPSHOT="/opt/etcd-backup-$(date +%Y%m%d_%H%M%S).db"
REMOTE_USER="backup-user"
REMOTE_HOST="192.168.2.100"
REMOTE_DIR="/backups/etcd"

# Take snapshot
ETCDCTL_API=3 etcdctl snapshot save "$SNAPSHOT" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key || exit 1

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status "$SNAPSHOT" --write-out=table

# Send to remote server
scp -i /root/.ssh/backup_key "$SNAPSHOT" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"

if [ $? -eq 0 ]; then
  echo "Remote backup successful: $SNAPSHOT"
  rm "$SNAPSHOT"    # clean up local copy after confirmed transfer
else
  echo "ALERT: Remote backup transfer failed!" >&2
  exit 1
fi
EOF

chmod +x /usr/local/bin/etcd-remote-backup.sh

# Schedule daily at 2 AM
echo "0 2 * * * root /usr/local/bin/etcd-remote-backup.sh >> /var/log/etcd-remote-backup.log 2>&1" \
  > /etc/cron.d/etcd-remote-backup
```
