# Q19. Restore etcd from Backup

## Task
Restore etcd from backup `/opt/etcd-backup-20240115.db` to data directory `/var/lib/etcd-restored`.
Update etcd manifest and verify cluster functionality.

---

## Phase 1: Stop the API Server (Prevent Conflicts)

```bash
# Move the API server manifest out — kubelet will stop it
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# Wait until API server pod stops
watch crictl ps
# Wait until kube-apiserver disappears
```

---

## Phase 2: Restore the Snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup-20240115.db \
  --data-dir=/var/lib/etcd-restored \
  --name=control-plane-1 \
  --initial-cluster=control-plane-1=https://192.168.1.100:2380 \
  --initial-advertise-peer-urls=https://192.168.1.100:2380 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

> For a single-node cluster, `--name` and `--initial-cluster` match the existing etcd member name.
> Check the existing name with: `ETCDCTL_API=3 etcdctl member list --endpoints=...`

---

## Phase 3: Update etcd Static Pod Manifest

```bash
# Point etcd to the restored data directory
vi /tmp/etcd.yaml
```

Find and change:
```yaml
# Before
- --data-dir=/var/lib/etcd

# After
- --data-dir=/var/lib/etcd-restored
```

Also update the `hostPath` volume mount:
```yaml
volumes:
- hostPath:
    path: /var/lib/etcd-restored    # changed from /var/lib/etcd
    type: DirectoryOrCreate
  name: etcd-data
```

---

## Phase 4: Restore Manifests and Restart

```bash
# Restore etcd first, then API server
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
# Wait for etcd pod to start
watch crictl ps

# Then restore API server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Wait for all control plane components
watch crictl ps
```

---

## Phase 5: Verify Cluster Functionality

```bash
# Give the cluster ~60s to fully restart
sleep 60

kubectl get nodes
kubectl get pods -A

# Verify etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

```
https://127.0.0.1:2379 is healthy: successfully committed proposal
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Forgot to update `hostPath` volume in manifest | Must update BOTH `--data-dir` arg AND hostPath volume |
| API server still running during restore | Stop it by moving manifest out first |
| Wrong `--name` in restore command | Match name from `etcdctl member list` |
| Ownership of restored dir is wrong | `chown -R etcd:etcd /var/lib/etcd-restored` |
