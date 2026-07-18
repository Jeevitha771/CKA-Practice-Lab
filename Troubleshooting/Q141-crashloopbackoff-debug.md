# Q141. Debug Pod in CrashLoopBackOff

## Task
Pod in `CrashLoopBackOff`. Investigate using logs, describe, events to identify root cause.

---

## Phase 1: Identify the Symptom

```bash
kubectl get pods
# NAME          READY   STATUS             RESTARTS   AGE
# crashed-app   0/1     CrashLoopBackOff   5          8m
```

`CrashLoopBackOff` means:
- Container starts → crashes → Kubernetes restarts it
- This cycle repeats with exponential backoff (10s, 20s, 40s, 80s...)

---

## Phase 2: Check Logs from the Crashed Container

```bash
# Current attempt (may be empty if crashed immediately)
kubectl logs crashed-app

# PREVIOUS crashed instance — most useful!
kubectl logs crashed-app -p
```

Common crash outputs:
```
# Missing environment variable
panic: runtime error: invalid memory address
Error: DATABASE_URL environment variable not set

# Port already in use
listen tcp :8080: bind: address already in use

# Missing config file
open /etc/app/config.yaml: no such file or directory

# OOMKilled (not in logs — see describe)
```

---

## Phase 3: Describe the Pod

```bash
kubectl describe pod crashed-app
```

Key sections to check:

```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error           ← or OOMKilled, Completed
  Exit Code:    1               ← non-zero = error; 137 = OOMKilled; 143 = SIGTERM
  Started:      ...
  Finished:     ...

Liveness:  http-get http://:8080/health delay=5s ...
           1 probe(s) failed

Events:
  Warning  BackOff    2m    kubelet  Back-off restarting failed container
  Warning  Failed     5m    kubelet  Error: failed to start container
```

---

## Phase 4: Common Root Causes and Fixes

### Cause 1: Missing Environment Variable / ConfigMap / Secret

```bash
kubectl describe pod crashed-app | grep -A 5 "Environment:"
# DATABASE_URL: <set to the key 'url' in secret 'db-secret'> Optional: false

# Check if secret exists
kubectl get secret db-secret
# Error from server (NotFound) ← This is the problem

# Fix: create the secret
kubectl create secret generic db-secret --from-literal=url=postgres://...
```

### Cause 2: Wrong Image or Tag

```bash
kubectl describe pod crashed-app | grep Image:
# Image: my-app:lates  ← typo

# Fix: update the image
kubectl set image pod/crashed-app my-app=my-app:latest
```

### Cause 3: Liveness Probe Failing Too Quickly

```bash
# If exit code is 0 but still restarting, liveness probe is killing it
kubectl describe pod crashed-app | grep -A 5 Liveness

# Fix: increase initial delay
kubectl edit deployment <deployment-name>
# Change initialDelaySeconds: 5 → 30
```

### Cause 4: OOMKilled (Exit Code 137)

```bash
kubectl describe pod crashed-app | grep -i oom
# OOMKilled  ← container hit memory limit

# Fix: increase memory limit
kubectl set resources deployment/<name> --limits=memory=512Mi
```

---

## Phase 5: Check Events

```bash
kubectl get events --sort-by='.lastTimestamp' | grep crashed-app
# Warning  BackOff  Failed  CrashLoopBackOff
```

---

## CrashLoopBackOff Exit Codes

| Exit Code | Meaning |
|---|---|
| `0` | Clean exit (but liveness probe failing) |
| `1` | Application error (check logs) |
| `137` | OOMKilled (SIGKILL) |
| `139` | Segfault |
| `143` | SIGTERM (graceful kill, timeout exceeded) |
