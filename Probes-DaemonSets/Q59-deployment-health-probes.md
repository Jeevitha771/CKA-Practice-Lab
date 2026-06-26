# Q59. Create Deployment with Liveness, Readiness, and Startup Probes

## Task
Create Deployment with health checks:
- Liveness: HTTP GET `/health:8080`, initialDelaySeconds=15, periodSeconds=10
- Readiness: HTTP GET `/ready:8080`, initialDelaySeconds=5, periodSeconds=5
- Startup: HTTP GET `/startup:8080`, failureThreshold=30, periodSeconds=10

## Probe Lifecycle

```
Container Starts
      |
      v
Startup Probe Runs  (shields liveness & readiness while running)
      |
      +--> Fail -> Keep retrying (30 * 10s = max 300s to boot)
      |
      +--> Success (startup probe disables itself permanently)
                |
                v
       Liveness Probe Starts   (kills & restarts container if fails)
       Readiness Probe Starts  (removes from Service endpoints if fails)
```

## Solution YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-dep-health
  labels:
    app: my-dep-health
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-dep-health
  template:
    metadata:
      labels:
        app: my-dep-health
    spec:
      containers:
      - name: nginx
        image: nginx

        startupProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 30   # 30 * 10s = 300s max startup time
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5       # Removes pod from service if fails (no restart)

        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10      # Kills and restarts container if fails
```

## Probe Types

| Type | Method | Example |
|---|---|---|
| HTTP GET | `httpGet` | `path: /health, port: 8080` |
| TCP Socket | `tcpSocket` | `port: 5432` |
| Exec Command | `exec` | `command: ["cat", "/tmp/healthy"]` |

## Key Differences

| Probe | On Failure | Use For |
|---|---|---|
| `startupProbe` | Kills container (same as liveness) | Slow-starting apps |
| `livenessProbe` | Kills and restarts container | Detecting deadlocks, frozen apps |
| `readinessProbe` | Removes from Service endpoints (**no restart**) | Waiting for DB connections, warming up |

## Notes
- Readiness failure = **stop traffic**, not restart
- Liveness failure = **restart container**
- startupProbe disables liveness+readiness while it is running
