# Q78. Use `kubectl diff` to Preview Changes Before Applying

## Task
Use `kubectl diff` to preview changes before applying a modified manifest.
Demonstrate with a deployment update.

## Solution

### Step 1: Deploy Initial Version

```yaml
# deployment-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

```bash
kubectl apply -f deployment-v1.yaml
kubectl get deployment web-app
```

### Step 2: Modify the File

Edit `deployment-v1.yaml`:
- Change `replicas: 2` → `replicas: 5`
- Change `image: nginx:1.21` → `image: nginx:1.22`

### Step 3: Preview Changes with kubectl diff

```bash
kubectl diff -f deployment-v1.yaml
```

**Output (similar to git diff):**
```diff
-  replicas: 2
+  replicas: 5
...
-        image: nginx:1.21
+        image: nginx:1.22
```

Lines starting with `-` = what's currently in the cluster  
Lines starting with `+` = what will be applied from the file

### Step 4: Apply After Confirming

```bash
kubectl apply -f deployment-v1.yaml
kubectl rollout status deployment/web-app
```

## When to Use kubectl diff

| Scenario | Tool |
|---|---|
| See what will change before applying | `kubectl diff -f <file>` |
| Validate syntax without connecting | `kubectl apply --dry-run=client` |
| Full server-side validation | `kubectl apply --dry-run=server` |
| View live resource | `kubectl get -o yaml` or `kubectl describe` |

## Notes
- `kubectl diff` requires the resource to already exist in the cluster
- Uses the same diff format as `git diff` (red = removed, green = added)
- Safe to run anytime — makes no changes
- Exit code `0` = no differences; exit code `1` = differences found (useful in CI/CD pipelines)
