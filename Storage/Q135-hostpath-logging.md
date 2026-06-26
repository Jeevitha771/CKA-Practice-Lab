# Q135. Configure Pod Using hostPath Volume for Logging — Security Implications

## Task
Configure pod using hostPath volume for logging.
Explain security implications and alternatives.

## hostPath Architecture

```
Node Filesystem
┌──────────────────────┐
│ /var/log/myapp/      │
└──────────┬───────────┘
           │ (same data, shared)
           ▼
Pod Container
┌──────────────────────┐
│ /logs                │
└──────────────────────┘
```

## Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: logging-pod
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: log-volume
      mountPath: /logs

  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/myapp
      type: DirectoryOrCreate  # Creates dir if it doesn't exist
```

```bash
kubectl apply -f pod.yaml

# Write a log entry inside pod
kubectl exec -it logging-pod -- sh
echo "Application started" > /logs/app.log
exit

# Verify log exists on the host node
ls /var/log/myapp
# app.log
```

## hostPath Types

| Type | Behaviour |
|---|---|
| `Directory` | Directory must already exist on host |
| `DirectoryOrCreate` | Creates directory if absent |
| `File` | File must already exist on host |
| `FileOrCreate` | Creates file if absent |

## Security Implications

⚠️ **Risks of hostPath:**

| Risk | Detail |
|---|---|
| Node filesystem access | Container can read/write host files — data exfiltration risk |
| Privilege escalation | Malicious pod can access sensitive host paths (e.g. `/etc`) |
| No portability | Pod is tied to a specific node; won't work on other nodes |
| Not suitable for multi-replica | Different pods on different nodes see different data |

## Safer Alternatives

| Alternative | Use Case |
|---|---|
| `emptyDir` | Temporary scratch space within a pod |
| `PVC + PV` | Persistent, portable storage |
| `Sidecar log shipper` | Ship logs to Elasticsearch/Fluentd without hostPath |
| `DaemonSet log agent` | Node-level log collection (Fluentd, Filebeat) |

## Notes
- `hostPath` is acceptable for practice/lab environments
- In production, prefer PVCs or log shipping sidecars
- If hostPath is required, use `readOnly: true` to reduce risk
