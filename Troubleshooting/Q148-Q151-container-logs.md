# Q148–Q151. Container Log Access and Debugging

## Q148 Task
Stream logs from pod in real-time and filter for ERROR messages.

## Q149 Task
Retrieve logs from multi-container pod, specifically from sidecar container.

## Q150 Task
Configure pod to write logs to both stdout and file in persistent volume.

## Q151 Task
Pod logs not appearing. Debug: check container status, log driver config, disk space on node, log rotation settings.

---

## Q148: Stream and Filter Logs

```bash
# Stream logs in real-time
kubectl logs -f my-pod

# Filter for ERROR messages (pipe to grep)
kubectl logs -f my-pod | grep -i error

# Filter for ERROR with context (3 lines after each match)
kubectl logs my-pod | grep -A 3 "ERROR"

# Since a time period + filter
kubectl logs --since=1h my-pod | grep -i "exception\|error\|fatal"

# Tail last 200 lines and stream
kubectl logs -f --tail=200 my-pod
```

---

## Q149: Multi-Container Pod — Specific Container Logs

```bash
# List containers in the pod first
kubectl get pod multi-pod -o jsonpath='{.spec.containers[*].name}'
# main-app sidecar

# Get logs from sidecar container specifically
kubectl logs multi-pod -c sidecar

# Stream sidecar logs
kubectl logs -f multi-pod -c sidecar

# Previous crashed sidecar
kubectl logs multi-pod -c sidecar -p
```

---

## Q150: Pod Writing Logs to stdout AND File in PVC

```yaml
# pod-dual-logging.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: dual-log-pod
spec:
  containers:
  - name: app
    image: busybox:1.28
    command:
    - sh
    - -c
    - |
      while true; do
        MSG="$(date) INFO: running"
        echo "$MSG"                          # stdout → kubectl logs
        echo "$MSG" >> /var/log/app/app.log  # file → PVC
        sleep 5
      done
    volumeMounts:
    - name: log-storage
      mountPath: /var/log/app
  volumes:
  - name: log-storage
    persistentVolumeClaim:
      claimName: log-pvc
```

```bash
kubectl apply -f pod-dual-logging.yaml

# Stream stdout
kubectl logs -f dual-log-pod

# Check file in PVC
kubectl exec dual-log-pod -- tail -f /var/log/app/app.log
```

---

## Q151: Pod Logs Not Appearing — Debug

### Check 1: Container is Running

```bash
kubectl get pod silent-pod
# STATUS: Running ← is it actually running?

kubectl describe pod silent-pod | grep -A 3 "State:"
# State: Waiting — pod isn't actually running yet
```

### Check 2: Is the App Writing to stdout?

```bash
# Some apps write logs to files, not stdout
# kubectl logs only captures stdout/stderr

kubectl exec silent-pod -- ls /var/log/
kubectl exec silent-pod -- cat /var/log/app.log  # may be here instead
```

### Check 3: Disk Space on Node

```bash
# SSH to the node hosting the pod
ssh <node>
df -h /var/log
# If /var/log is full, logs can't be written

# Find large log files
du -sh /var/log/pods/* | sort -h | tail -10
```

### Check 4: Log Rotation Settings

```bash
# kubelet log rotation
cat /var/lib/kubelet/config.yaml | grep -i log
# containerLogMaxSize: 10Mi      # rotate at 10Mi
# containerLogMaxFiles: 5        # keep 5 rotated files

# Check actual log files
ls -la /var/log/pods/<namespace>_<pod>_<uid>/<container>/
```

### Check 5: Restart kubelet if Log Driver Broken

```bash
systemctl restart kubelet
kubectl logs silent-pod
```
