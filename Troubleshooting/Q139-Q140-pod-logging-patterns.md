# Q139–Q140. Pod Logging — All Patterns

## Q139 Task
Access and analyze logs for:
- Specific pod
- Previous crashed pod instance
- All pods with a specific label
- kubelet on a node

## Q140 Task
Show logs from multiple containers in a pod simultaneously.

---

## Q139: All Logging Patterns

### Current Pod Logs

```bash
# Basic pod logs
kubectl logs <pod-name>

# Specific namespace
kubectl logs <pod-name> -n <namespace>

# Follow (stream) in real time
kubectl logs -f <pod-name>

# Last N lines
kubectl logs --tail=100 <pod-name>

# Logs from last 1 hour
kubectl logs --since=1h <pod-name>

# Combine: follow last 50 lines since 30 minutes
kubectl logs -f --tail=50 --since=30m <pod-name>
```

---

### Previous Crashed Pod Instance

```bash
# -p / --previous flag shows logs from the LAST crashed container
kubectl logs <pod-name> -p

# Example: pod has restarted 3 times
kubectl get pod crashed-pod
# NAME           READY   STATUS             RESTARTS
# crashed-pod    0/1     CrashLoopBackOff   3

kubectl logs crashed-pod -p
# Shows last container's logs before it died
```

---

### All Pods with a Specific Label

```bash
# -l selector logs from ALL matching pods
kubectl logs -l app=web-frontend --all-containers

# With namespace
kubectl logs -l app=api-server -n production --tail=50

# Note: works across a selector; good for debugging a deployment
```

---

### kubelet Logs (on the Node)

```bash
# SSH to the node first
ssh <node-name>

# View kubelet systemd service logs
journalctl -u kubelet
journalctl -u kubelet -n 100 --no-pager   # last 100 lines
journalctl -u kubelet --since "1 hour ago"
journalctl -u kubelet -f                  # follow live

# Filter for errors only
journalctl -u kubelet | grep -i error
```

---

## Q140: Multi-Container Pod Logging

### List Containers in a Pod

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].name}'
# main-app sidecar logger
```

### Log from a Specific Container

```bash
kubectl logs <pod-name> -c <container-name>
kubectl logs multi-pod -c sidecar
```

### Log from ALL Containers in a Pod

```bash
kubectl logs <pod-name> --all-containers=true
kubectl logs <pod-name> --all-containers=true --prefix=true
# [pod/multi-pod/main-app] Starting server...
# [pod/multi-pod/sidecar] Received request
```

### Simulate with tmux (Multiple Terminals)

```bash
# Terminal 1: main app logs
kubectl logs -f multi-pod -c main-app

# Terminal 2: sidecar logs
kubectl logs -f multi-pod -c sidecar
```

### stern (Third-Party Tool — Not Available in Exam)

```bash
# For reference only — not in CKA exam
stern <pod-name>
```

---

## Quick Reference

| Goal | Command |
|---|---|
| Current logs | `kubectl logs <pod>` |
| Previous crash | `kubectl logs <pod> -p` |
| Follow live | `kubectl logs -f <pod>` |
| By label | `kubectl logs -l app=x --all-containers` |
| Specific container | `kubectl logs <pod> -c <container>` |
| All containers | `kubectl logs <pod> --all-containers=true --prefix=true` |
| kubelet logs | `journalctl -u kubelet -n 100` |
