# Q143–Q145. Resource Monitoring with kubectl top

## Q143 Task
Check resource usage (CPU/memory) for: all nodes, all pods in namespace, specific pod, containers in pod.

## Q144 Task
Identify pods consuming most CPU and memory. Recommend optimizations.

## Q145 Task
Configure resource metrics collection. Verify `kubectl top` commands work.

---

## Q145: Install and Verify Metrics Server

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics-server to be ready
kubectl rollout status deployment metrics-server -n kube-system

# If in a local lab (kubeadm/minikube), add --kubelet-insecure-tls:
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Verify kubectl top works
kubectl top nodes
kubectl top pods -A
```

---

## Q143: Check Resource Usage

### All Nodes

```bash
kubectl top nodes
```
```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
control-plane  312m         15%    1523Mi          40%
worker-1       89m          4%     752Mi           20%
worker-2       1420m        71%    3012Mi          79%    ← high usage
```

---

### All Pods in a Namespace

```bash
kubectl top pods -n production
```
```
NAME              CPU(cores)   MEMORY(bytes)
api-server-abc    250m         192Mi
web-frontend-xyz  45m          85Mi
database-0        880m         1400Mi           ← high
```

---

### Specific Pod

```bash
kubectl top pod api-server-abc -n production
```
```
NAME              CPU(cores)   MEMORY(bytes)
api-server-abc    250m         192Mi
```

---

### Containers Within a Pod

```bash
kubectl top pod multi-container-pod --containers
```
```
POD                    NAME        CPU(cores)   MEMORY(bytes)
multi-container-pod    main-app    200m         150Mi
multi-container-pod    sidecar     50m          42Mi
```

---

## Q144: Identify Top Resource Consumers

### Top CPU-consuming Pods

```bash
# Sort by CPU (descending)
kubectl top pods -A --sort-by=cpu | head -10
```
```
NAMESPACE    NAME            CPU(cores)   MEMORY(bytes)
production   database-0      880m         1400Mi
kube-system  etcd-master     210m         180Mi
production   api-server-abc  250m         192Mi
```

---

### Top Memory-consuming Pods

```bash
kubectl top pods -A --sort-by=memory | head -10
```

---

### Top Node Resource Usage

```bash
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory
```

---

## Optimization Recommendations

| Finding | Recommendation |
|---|---|
| Pod using > 80% of its CPU limit | Increase CPU limit or optimize code |
| Pod CPU >> CPU request | Increase request to get better scheduling |
| Pod OOMKilled frequently | Increase memory limit |
| Node memory > 80% | Add more nodes or reschedule pods |
| Idle pods with high allocations | Reduce requests/limits to reclaim resources |

```bash
# Check what resources are requested vs actual usage
kubectl describe node worker-1 | grep -A 10 "Allocated resources"
# Shows requested vs available — a pod can have high requests but low actual usage
```
