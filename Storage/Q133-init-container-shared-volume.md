# Q133. Pod with Init Container Preparing Data in Shared Volume

## Task
Create pod with init container that prepares data in a shared volume before the main container starts.

## Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-pod
  labels:
    app: init-demo
spec:
  # Shared volume between init and main container
  volumes:
  - name: shared-data
    emptyDir: {}

  # Init container runs first — must complete before main container starts
  initContainers:
  - name: data-prep
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Data successfully prepared by the init container!" > /work/index.txt']
    volumeMounts:
    - name: shared-data
      mountPath: /work

  # Main container starts only after init container exits successfully
  containers:
  - name: main-app
    image: nginx:alpine
    command: ['sh', '-c', 'cat /work/index.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /work
```

```bash
kubectl apply -f pod.yaml

# Watch init container complete, then main container start
kubectl get pod init-demo-pod -w

# Verify data was prepared
kubectl exec init-demo-pod -c main-app -- cat /work/index.txt
# Output: Data successfully prepared by the init container!
```

## How Init Containers Work

```
Init container (data-prep) runs
        ↓
Writes file to /work (shared emptyDir volume)
        ↓
Init container exits with code 0
        ↓
Main container (main-app) starts
        ↓
Reads file from /work (same shared volume)
```

## Key Rules

| Rule | Detail |
|---|---|
| Init containers run sequentially | Each must complete before the next starts |
| Main containers start after ALL inits complete | Only if all inits exit with code 0 |
| Init containers share volumes with main containers | via `emptyDir` or PVC |
| If init container fails | Pod restarts based on `restartPolicy` |

## Restart Policies

| Policy | Behaviour on Init Failure |
|---|---|
| `Always` | Pod keeps restarting init container until it succeeds |
| `OnFailure` | Restart only if exit code is non-zero |
| `Never` | Pod fails permanently if init container fails |

## Notes
- `-c <container-name>` flag in `kubectl exec` is needed when Pod has multiple containers
- Init containers are commonly used for: DB migrations, config downloads, dependency waits
- `emptyDir` is perfect for sharing data between init and main containers within same pod
