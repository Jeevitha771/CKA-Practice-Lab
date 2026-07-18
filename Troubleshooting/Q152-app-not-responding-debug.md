# Q152. Systematically Debug Application Not Responding

## Task
Application not responding. Systematically debug: pod status/events, container logs, liveness/readiness probes, resource constraints, network connectivity, ConfigMaps/Secrets.

---

## Systematic Debug Framework (Work Outward)

```
Pod → Container → App → Network → Config
```

---

## Step 1: Check Pod Status and Events

```bash
kubectl get pods -n <namespace>
# NAME       READY   STATUS    RESTARTS   AGE
# app-pod    0/1     Running   0          5m   ← READY 0/1 = failing probes

kubectl describe pod app-pod -n <namespace>
# Read carefully:
#   - State / Reason / Exit Code
#   - Conditions (Ready, ContainersReady)
#   - Events (bottom of output — most recent issues)
```

---

## Step 2: Check Container Logs

```bash
# Current logs
kubectl logs app-pod -n <namespace>

# Previous crashed instance
kubectl logs app-pod -n <namespace> -p

# Filter for errors
kubectl logs app-pod -n <namespace> | grep -i "error\|fatal\|panic"
```

---

## Step 3: Check Liveness / Readiness Probes

```bash
kubectl describe pod app-pod -n <namespace> | grep -A 10 "Liveness\|Readiness"
```

```
Liveness:   http-get http://:8080/health delay=5s timeout=1s period=10s #failure=3
Readiness:  http-get http://:8080/ready delay=5s timeout=1s period=5s #failure=3
```

```bash
# Test probe manually from inside the pod
kubectl exec app-pod -n <namespace> -- wget -qO- http://localhost:8080/health
# If this fails, the probe endpoint is broken
```

---

## Step 4: Check Resource Constraints

```bash
kubectl describe pod app-pod -n <namespace> | grep -A 6 "Limits\|Requests"

# Is the pod being throttled by CPU?
kubectl top pod app-pod -n <namespace>
# NAME      CPU(cores)   MEMORY(bytes)
# app-pod   498m         200Mi    ← near CPU limit of 500m = throttled

# Is it OOMKilled?
kubectl describe pod app-pod | grep -i oomkilled
```

---

## Step 5: Check Network Connectivity

```bash
# Can the pod reach its dependencies?
kubectl exec app-pod -n <namespace> -- wget -qO- --timeout=3 http://database-service:5432
kubectl exec app-pod -n <namespace> -- nslookup database-service

# Check if the service has endpoints
kubectl get endpoints <service-name> -n <namespace>
# If ENDPOINTS is empty: no pods match the selector

# Check NetworkPolicies
kubectl get networkpolicy -n <namespace>
```

---

## Step 6: Check ConfigMaps and Secrets

```bash
# Verify required ConfigMaps exist
kubectl get configmap -n <namespace>

# Check if a ConfigMap key is missing
kubectl describe pod app-pod | grep -A 3 "Environment:"
# DATABASE_URL:  <set to the key 'url' in secret 'db-secret'>  Optional: false

# Does the secret exist and have the key?
kubectl get secret db-secret -n <namespace> -o jsonpath='{.data}' | python3 -m json.tool

# Check volume mounts
kubectl describe pod app-pod | grep -A 10 "Mounts:\|Volumes:"
```

---

## Step 7: Interactive Debug Shell

```bash
# Exec into the running container for live investigation
kubectl exec -it app-pod -n <namespace> -- /bin/sh

# Inside: check environment, connectivity, filesystem
env | grep DATABASE
wget -qO- http://localhost:8080/health
cat /etc/app/config.yaml
```

---

## Summary Checklist

| Layer | Command | Look For |
|---|---|---|
| Pod state | `kubectl get pod` | Running and 1/1 Ready |
| Events | `kubectl describe pod` | Warning events |
| Logs | `kubectl logs [-p]` | Error/panic messages |
| Probes | `kubectl describe` → exec probe | Returns 2xx |
| Resources | `kubectl top pod` | Not at 100% limit |
| Network | `kubectl get endpoints` | Populated, not empty |
| Config | `kubectl get secret/cm` | Exists and has correct keys |
