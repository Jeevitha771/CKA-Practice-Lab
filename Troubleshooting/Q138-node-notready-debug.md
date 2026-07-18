# Q138. Debug Node Reporting NotReady

## Task
Node reporting `NotReady`. Debug: check node conditions, kubelet logs, container runtime status, system resources (CPU/memory/disk).

---

## Phase 1: Identify the Problem

```bash
kubectl get nodes
# NAME       STATUS     ROLES    AGE
# worker-1   NotReady   <none>   5d
```

---

## Phase 2: Check Node Conditions

```bash
kubectl describe node worker-1
```

Look at the `Conditions` section:
```
Conditions:
  Type                Status  Reason
  ----                ------  ------
  MemoryPressure      False
  DiskPressure        False
  PIDPressure         False
  Ready               False   KubeletNotReady  # ← The key condition
```

Also check `Events` at the bottom for recent messages.

---

## Phase 3: Check kubelet Logs (SSH to the Node)

```bash
ssh worker-1

# Check kubelet service status
systemctl status kubelet
# Active: activating (auto-restart) (Result: exit-code)

# View recent kubelet logs
journalctl -u kubelet -n 100 --no-pager
# Look for errors like:
#   "Unable to connect to Kubernetes API"
#   "failed to create containerd task"
#   "certificate has expired"
#   "No space left on device"
```

---

## Phase 4: Check Container Runtime

```bash
systemctl status containerd
# If not running:
systemctl restart containerd
systemctl status containerd

# Test crictl
crictl ps
# If it hangs — containerd socket issue
```

---

## Phase 5: Check System Resources

```bash
# CPU and memory
top -bn1 | head -10
free -h

# Disk space (disk pressure evicts pods and causes NotReady)
df -h
# If / or /var is >85%, clear space:
crictl rmi --prune            # remove unused images
journalctl --vacuum-size=500M # trim system logs

# Check inode usage
df -i | grep -v tmpfs

# Check PIDs
cat /proc/sys/kernel/pid_max
ps aux | wc -l
```

---

## Phase 6: Restart kubelet and Verify

```bash
systemctl restart kubelet
journalctl -u kubelet -f   # follow logs

# On control plane:
kubectl get nodes -w
# worker-1   Ready   ...   ← should recover
```

---

## Debug Checklist

| Check | Command | Healthy Sign |
|---|---|---|
| Node conditions | `kubectl describe node <name>` | All False except Ready=True |
| kubelet service | `systemctl status kubelet` | `active (running)` |
| kubelet logs | `journalctl -u kubelet -n 50` | No ERROR lines |
| containerd | `systemctl status containerd` | `active (running)` |
| Disk space | `df -h` | < 85% used |
| Memory | `free -h` | Available > 0 |
| CPU | `top -bn1` | Not 100% sustained |
| Swap | `swapon --show` | Empty (swap disabled) |
