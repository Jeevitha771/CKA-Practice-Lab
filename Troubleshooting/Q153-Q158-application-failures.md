# Q153–Q158. Application Failure Troubleshooting

## Q153 Task
Deployment rollout stuck. Identify why new pods aren't starting and fix.

## Q154 Task
Pods running but application returns 503 errors. Debug service, endpoints, ingress.

## Q155 Task
Application works locally but fails in Kubernetes. Debug env vars, volume mounts, network policies.

## Q156 Task
Pod stuck Pending. Identify and resolve scheduling issue.

## Q157 Task
Application cannot connect to database. Check: service DNS, network policies, DB pod status, credentials.

## Q158 Task
Application experiencing intermittent failures. Debug and identify root cause.

---

## Q153: Stuck Deployment Rollout

```bash
kubectl rollout status deployment/web-app -n production
# Waiting for deployment "web-app" rollout to finish: 1 out of 3 new replicas have been updated...

# Check what's happening with pods
kubectl get pods -n production
# NAME               READY   STATUS             RESTARTS
# web-app-old-abc    1/1     Running            0
# web-app-new-xyz    0/1     ImagePullBackOff   0   ← stuck here

kubectl describe pod web-app-new-xyz -n production
# Events: Failed to pull image "my-app:v2.0-typo": image not found
```

**Fixes:**
```bash
# Fix 1: Correct the image
kubectl set image deployment/web-app web-app=my-app:v2.0 -n production

# Fix 2: Rollback to working version
kubectl rollout undo deployment/web-app -n production

# Fix 3: Resource issue — check if node has capacity
kubectl describe pod web-app-new-xyz | grep -A 5 "Events:"
```

---

## Q154: Pods Running but 503 Errors

```bash
# Confirm pods are running and ready
kubectl get pods -n production -l app=web-app
# READY 1/1 = OK

# Step 1: Check service selector matches pod labels
kubectl get service web-service -n production -o yaml | grep selector
# selector:
#   app: webapp  ← typo!

kubectl get pods -n production --show-labels
# labels: app=web-app ← doesn't match "webapp"

# Fix: Update service selector
kubectl patch service web-service -n production \
  -p '{"spec":{"selector":{"app":"web-app"}}}'

# Step 2: Verify endpoints are populated
kubectl get endpoints web-service -n production
# NAME          ENDPOINTS
# web-service   10.244.1.5:8080,10.244.2.7:8080   ← now populated

# Step 3: Check Ingress backend config
kubectl describe ingress web-ingress -n production
# Backend: web-service:80 ← correct port?
```

---

## Q156: Pod Stuck Pending

```bash
kubectl describe pod stuck-pod
```

**Cause 1: Insufficient Resources**
```
Events: 0/2 nodes are available: 2 Insufficient cpu
```
```bash
# Check node available resources
kubectl describe nodes | grep -A 5 "Allocated resources"
# Reduce pod CPU request: kubectl edit pod stuck-pod
```

**Cause 2: Node Selector No Match**
```
Events: 0/2 nodes are available: 2 node(s) didn't match Pod's node affinity
```
```bash
kubectl describe pod stuck-pod | grep -A 5 "Node-Selectors\|Affinity"
# nodeSelector: disktype=ssd
kubectl get nodes --show-labels | grep disktype
# (no label) ← problem
kubectl label node worker-1 disktype=ssd
```

**Cause 3: Taint / Toleration Mismatch**
```
Events: 1 node(s) had taint {special: true}, that the pod didn't tolerate
```
```bash
kubectl describe nodes | grep Taints
# Taints: special=true:NoSchedule

# Add toleration to pod spec:
tolerations:
- key: special
  operator: Equal
  value: "true"
  effect: NoSchedule
```

---

## Q155: Works Locally, Fails in Kubernetes

```bash
# Check environment variables
kubectl exec failing-pod -- env | sort
# Compare to local: missing DATABASE_HOST?

# Check volume mounts
kubectl describe pod failing-pod | grep -A 10 "Mounts:"
# /config from config-volume (ro) — is the ConfigMap correct?
kubectl exec failing-pod -- cat /config/app.properties

# Check network policy
kubectl get networkpolicy -n <namespace>
# Is there a default deny blocking the app's outbound?

# Check DNS inside pod
kubectl exec failing-pod -- nslookup database-service
```

---

## Q157: Cannot Connect to Database

```bash
# Step 1: DNS resolution
kubectl exec app-pod -- nslookup database-service
# If fails: service name wrong or DNS broken

# Step 2: Service exists and has endpoints
kubectl get service database-service -n <namespace>
kubectl get endpoints database-service -n <namespace>
# ENDPOINTS: <none> ← db pod not running or label mismatch

# Step 3: DB pod status
kubectl get pods -n <namespace> -l app=database

# Step 4: Credentials from secret
kubectl exec app-pod -- env | grep DB_PASS
# Empty? Secret missing or key name wrong
kubectl get secret db-secret -n <namespace>
kubectl get secret db-secret -n <namespace> -o jsonpath='{.data.password}' | base64 -d

# Step 5: NetworkPolicy blocking
kubectl get networkpolicy -n <namespace>
# Is egress to DB port 5432 allowed?
```

---

## Q158: Intermittent Failures

```bash
# Check restart counts (intermittent = periodic crash)
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# Look at events for warning patterns
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20

# Check HPA (if scaling causes issues)
kubectl get hpa -n <namespace>
# New pods starting without readiness = intermittent errors during scale-up

# Memory leak? Check memory trending
kubectl top pod -n <namespace>
# Run repeatedly — does memory grow over time?

# Liveness probe too aggressive?
kubectl describe pod <pod> | grep -A 5 "Liveness:"
# Try reducing periodSeconds or increasing failureThreshold
```
