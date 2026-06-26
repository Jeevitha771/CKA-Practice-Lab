# Q64. Pod Keeps Restarting Due to Failed Liveness Probes — Fix Configuration

## Task
Pod keeps restarting due to failed liveness probes.
Adjust: `initialDelaySeconds`, `timeoutSeconds`, `failureThreshold`, `periodSeconds`.

## Problem — Broken Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo-broken
spec:
  containers:
  - name: slow-app
    image: busybox
    # App waits 15s before creating the file the probe checks for
    args:
    - /bin/sh
    - -c
    - "sleep 15; touch /tmp/healthy; sleep 3600"
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 2    # ❌ Too early — app not ready yet
      periodSeconds: 3
      failureThreshold: 2       # ❌ Kills after only 2 failures
      timeoutSeconds: 1
```

**What happens:** Probe fires at 2s → file doesn't exist (fail 1) → fires at 5s → still no file (fail 2) → Pod killed and restarted. Never makes it to 15s.

```bash
kubectl apply -f broken-pod.yaml
kubectl get pods -w              # Watch restart loop
kubectl describe pod liveness-demo-broken  # See liveness probe events
```

## Fix — Corrected Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo-fixed
spec:
  containers:
  - name: slow-app
    image: busybox
    args:
    - /bin/sh
    - -c
    - "sleep 15; touch /tmp/healthy; sleep 3600"
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 20   # ✅ Wait 20s before first check
      periodSeconds: 5
      failureThreshold: 3       # ✅ Allow 3 failures before restart
      timeoutSeconds: 2         # ✅ Wait 2s for response
```

## Better Fix — Use startupProbe (Best Practice)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-probe-demo
spec:
  containers:
  - name: slow-app
    image: busybox
    args:
    - /bin/sh
    - -c
    - "sleep 15; touch /tmp/healthy; sleep 3600"

    # SHIELD: Startup probe gives app up to 30s to boot
    startupProbe:
      exec:
        command: [cat, /tmp/healthy]
      failureThreshold: 10    # 10 * 3s = 30 seconds max
      periodSeconds: 3

    # ONGOING: Liveness is aggressive AFTER startup succeeds
    livenessProbe:
      exec:
        command: [cat, /tmp/healthy]
      failureThreshold: 2
      periodSeconds: 2
      initialDelaySeconds: 0
```

## Probe Configuration Fields

| Field | Purpose |
|---|---|
| `initialDelaySeconds` | Wait this long after container starts before first probe |
| `periodSeconds` | How often to probe |
| `timeoutSeconds` | How long to wait for probe response before marking as failed |
| `failureThreshold` | Consecutive failures before action is taken |

## Why startupProbe is Better

- `initialDelaySeconds` creates a **blind spot** — liveness is completely off during boot
- `startupProbe` is **transparent** — actively checking but patient about failures
- Once startupProbe succeeds once, it disables itself and hands off to liveness
