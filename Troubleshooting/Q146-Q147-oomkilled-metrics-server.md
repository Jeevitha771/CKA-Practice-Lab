# Q146–Q147. OOMKilled and Metrics Server Setup

## Q146 Task
Pod being OOMKilled. Analyze issue and adjust resource limits appropriately.

## Q147 Task
Set up metrics-server in the cluster and verify it's collecting metrics from all nodes.

---

## Q147: Set Up metrics-server

```bash
# Apply the official metrics-server manifest
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Monitor rollout
kubectl rollout status deployment/metrics-server -n kube-system
# deployment "metrics-server" successfully rolled out

# Verify pods running
kubectl get pods -n kube-system -l k8s-app=metrics-server
# NAME                           READY   STATUS    RESTARTS
# metrics-server-6d4d9c4d7-abc   1/1     Running   0
```

### Fix for Lab/kubeadm (TLS Certificate Issue)

```bash
# metrics-server needs to trust kubelet cert
# In local clusters, add --kubelet-insecure-tls flag:

kubectl edit deployment metrics-server -n kube-system
```

Add under `args:`:
```yaml
args:
- --cert-dir=/tmp
- --secure-port=4443
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
- --kubelet-use-node-status-port
- --metric-resolution=15s
- --kubelet-insecure-tls    # ← add this line
```

```bash
# Verify metrics are collected from all nodes
kubectl top nodes
# NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# control-plane  410m         20%    2048Mi          54%
# worker-1       95m          5%     810Mi           21%
# (all nodes appear = metrics-server working)
```

---

## Q146: Debug and Fix OOMKilled Pod

### Step 1: Identify the OOMKilled Pod

```bash
kubectl get pods -A | grep -v Running
# NAMESPACE    NAME       READY   STATUS      RESTARTS
# production   api-pod    0/1     OOMKilled   7
```

### Step 2: Confirm OOM in Pod Description

```bash
kubectl describe pod api-pod -n production
```

```
Last State: Terminated
  Reason:    OOMKilled       ← memory limit exceeded
  Exit Code: 137             ← SIGKILL signal
  Started:   ...
  Finished:  ...

Limits:
  memory: 128Mi              ← too low
Requests:
  memory: 64Mi
```

### Step 3: Check Actual Memory Usage

```bash
# See current memory usage vs limit
kubectl top pod api-pod -n production
# NAME      CPU(cores)   MEMORY(bytes)
# api-pod   100m         125Mi         ← near 128Mi limit → getting killed
```

### Step 4: Increase Memory Limit

```bash
# Update the deployment's memory limit
kubectl set resources deployment/api-app \
  -n production \
  --limits=memory=512Mi \
  --requests=memory=256Mi

# Verify rollout
kubectl rollout status deployment/api-app -n production

# Confirm new pod has new limits
kubectl describe pod $(kubectl get pod -n production -l app=api-app -o name | head -1) \
  -n production | grep -A 4 Limits
```

### Step 5: Monitor to Confirm Fix

```bash
# Watch for OOMKill over time
kubectl get pods -n production -w
# Should now stay Running

# Check resource usage trending
kubectl top pod -n production --containers
```

---

## Analyzing OOM Patterns

```bash
# Check restart count — high restarts often = OOM loop
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# Get last termination reason for all pods
kubectl get pods -A -o json | python3 -c "
import json, sys
pods = json.load(sys.stdin)
for pod in pods['items']:
    for cs in pod.get('status', {}).get('containerStatuses', []):
        last = cs.get('lastState', {}).get('terminated', {})
        if last.get('reason') == 'OOMKilled':
            print(pod['metadata']['namespace'], pod['metadata']['name'], 'OOMKilled')
"
```
