# Q130. Create a PVC That Binds to a Specific PV by Name Using a Selector

## Task
Create a PVC that uses a specific PV by name using a selector.

## Method 1: Selector (match by label)

### PV with label

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: ssd                  # Label used for selector matching
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-storage
  hostPath:
    path: /tmp/data
```

### PVC with selector

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: fast-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      type: ssd                # Matches PV label
```

## Method 2: volumeName (exact binding — strict)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  volumeName: my-pv            # Direct binding to a specific PV by name
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

## Comparison

| Method | Behaviour |
|---|---|
| `selector.matchLabels` | Binds to any PV with matching labels (flexible) |
| `volumeName` | Binds to exact named PV (strict, guaranteed) |

## How Kubernetes Binds PVC to PV

All of these conditions must match:
- `storageClassName` ✅
- `accessModes` ✅
- PV `capacity` >= PVC `request` ✅
- `selector` labels (if specified) ✅

## Notes
- `selector` in PVC → must match `labels` in PV metadata
- Without a selector, Kubernetes picks any suitable PV automatically
- If selector is set but no PV matches → PVC stays in `Pending`
- One PVC binds to exactly one PV (1:1)
- Multiple Pods can share one PVC (subject to access mode)
