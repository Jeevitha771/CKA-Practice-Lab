# Q18. Backup etcd Cluster with etcdctl

## Task
Backup etcd cluster to `/opt/etcd-backup-$(date +%Y%m%d).db`.
- etcd runs as a static pod with certificates in `/etc/kubernetes/pki/etcd/`
- Verify backup integrity after creating it

---

## Phase 1: Find etcd Connection Details

```bash
# Check the etcd static pod manifest for endpoints and cert paths
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'listen-client|cert-file|key-file|trusted-ca'
```

Typical output:
```
--listen-client-urls=https://127.0.0.1:2379,https://192.168.1.100:2379
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

---

## Phase 2: Take the Snapshot

```bash
BACKUP_FILE="/opt/etcd-backup-$(date +%Y%m%d).db"

ETCDCTL_API=3 etcdctl snapshot save "$BACKUP_FILE" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

Output:
```
Snapshot saved at /opt/etcd-backup-20240116.db
```

---

## Phase 3: Verify Backup Integrity

```bash
ETCDCTL_API=3 etcdctl snapshot status "$BACKUP_FILE" \
  --write-out=table
```

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| a4e8b6c1 |    14523 |        943 |     3.2 MB |
+----------+----------+------------+------------+
```

Also verify the file exists:
```bash
ls -lh /opt/etcd-backup-*.db
```

---

## Convenience: Set etcdctl Alias

```bash
# Avoid repeating cert flags every command
alias etcdctl='ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

etcdctl snapshot save /opt/etcd-backup-$(date +%Y%m%d).db
etcdctl snapshot status /opt/etcd-backup-$(date +%Y%m%d).db --write-out=table
```

---

## etcdctl Certificate Flags Quick Reference

| Flag | Path |
|---|---|
| `--cacert` | `/etc/kubernetes/pki/etcd/ca.crt` |
| `--cert` | `/etc/kubernetes/pki/etcd/server.crt` |
| `--key` | `/etc/kubernetes/pki/etcd/server.key` |
| `--endpoints` | `https://127.0.0.1:2379` |

> 💡 **Exam tip**: The cert paths and endpoint are always visible in the etcd static pod manifest at `/etc/kubernetes/manifests/etcd.yaml`
