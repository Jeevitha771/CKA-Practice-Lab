# Q45. Revision History Limit — Retain Last 5 Revisions

## Task
- Create namespace `update-test`
- Create Deployment `web-updater` with image nginx:1.20, 3 replicas
- Configure Deployment to retain maximum **5** old ReplicaSets
- Trigger at least **8** rolling updates
- Verify revision history limit is functioning
- Redirect final ReplicaSet list to `/opt/outputs/web-updater-rs.txt`

## Solution

```bash
# Step 1: Create namespace
kubectl create ns update-test

# Step 2: Generate deployment YAML
kubectl create deploy web-updater \
  --image=nginx:1.20 \
  --replicas=3 \
  -n update-test \
  --dry-run=client -o yaml > deploy.yaml
```

### deploy.yaml — add revisionHistoryLimit

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-updater
  namespace: update-test
spec:
  replicas: 3
  revisionHistoryLimit: 5    # Retain only last 5 ReplicaSets
  selector:
    matchLabels:
      app: web-updater
  template:
    metadata:
      labels:
        app: web-updater
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

```bash
kubectl apply -f deploy.yaml

# Step 3: Trigger 8 rolling updates using env var changes
for i in {1..8}; do
  kubectl set env deploy web-updater -n update-test UPDATE_ID=$i
  sleep 1
done

# Step 4: Check revision history (should show max 5 + current = 6 entries)
kubectl rollout history deployment/web-updater -n update-test

# Step 5: Output ReplicaSets to file
mkdir -p /opt/outputs
kubectl get rs -n update-test > /opt/outputs/web-updater-rs.txt
cat /opt/outputs/web-updater-rs.txt
```

## Key Field

```bash
kubectl explain deployment.spec | grep -i limit
# revisionHistoryLimit
```

## Notes
- `revisionHistoryLimit: 5` keeps only the last 5 inactive ReplicaSets
- Default value is 10 if not set
- Use `kubectl patch` to update on a running deployment:
  `kubectl patch deployment web-updater -n update-test -p '{"spec":{"revisionHistoryLimit":5}}'`
