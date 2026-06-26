# Q77. Use `kubectl apply --dry-run=client` vs `--dry-run=server` to Validate Manifests

## Task
Use `kubectl apply --dry-run=client` and `--dry-run=server` to validate manifests.
Explain the difference.

## The Difference

| Mode | What It Does | Talks to Cluster? |
|---|---|---|
| `--dry-run=client` | Validates YAML structure and syntax locally | ❌ No |
| `--dry-run=server` | Sends to API server — runs admission controllers, checks dependencies | ✅ Yes (but doesn't save) |

## Demo Files

### Valid Pod

```yaml
# valid-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
```

### Tricky Pod (non-existent namespace)

```yaml
# invalid-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: trick-pod
  namespace: totally-fake-namespace   # This namespace does not exist
spec:
  containers:
  - name: nginx
    image: nginx:alpine
```

## Test client dry-run (catches syntax only)

```bash
# Valid pod — passes both
kubectl apply -f valid-pod.yaml --dry-run=client
# Output: pod/test-pod configured (dry run)

# Invalid pod — client dry-run PASSES (doesn't check namespace)
kubectl apply -f invalid-pod.yaml --dry-run=client
# Output: pod/trick-pod configured (dry run)  ← false positive!
```

## Test server dry-run (catches real errors)

```bash
# Invalid pod — server dry-run FAILS
kubectl apply -f invalid-pod.yaml --dry-run=server
# Error: namespaces "totally-fake-namespace" not found  ← caught!

# Valid pod — see what cluster would add (defaults, serviceAccount, tolerations)
kubectl apply -f valid-pod.yaml --dry-run=server -o yaml
# Shows: auto-added serviceAccount, tolerationSeconds, status block, etc.
```

## What Server Dry-Run Catches

- Namespace doesn't exist
- RBAC violations (no permission)
- Admission webhook rejections
- Custom validation rules
- Shows all defaulted fields the cluster would inject

## Exam Use Cases

```bash
# Quick syntax check before applying
kubectl apply -f manifest.yaml --dry-run=client

# Full validation before a risky change
kubectl apply -f manifest.yaml --dry-run=server

# Generate + immediately validate
kubectl create deployment api --image=nginx --dry-run=server -o yaml
```
