# Q82. Create Pod Using hostNetwork — Verify Node IP, Security Implications, Use Cases

## Task
Create pod using `hostNetwork: true`. Verify pod uses node IP. Explain security implications and list use cases.

## Solution

### Step 1: Check the Node's IP (for comparison)

```bash
kubectl get nodes -o wide
# Note the INTERNAL-IP of the node (e.g., 192.168.1.10)
```

### Step 2: Deploy hostNetwork Pod

```yaml
# host-net-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-host-network
spec:
  hostNetwork: true       # Pod shares the node's network namespace
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f host-net-pod.yaml
```

### Step 3: Deploy a Normal Pod for Comparison

```bash
kubectl run pod-normal-network --image=nginx
```

### Step 4: Verify and Compare

```bash
kubectl get pods -o wide

# pod-normal-network  IP: 10.244.x.x  (isolated Pod CIDR address)
# pod-host-network    IP: 192.168.1.10 (SAME as node INTERNAL-IP) ✅
```

The `pod-host-network` IP is **identical to the node's IP** — it is sharing the host's network stack.

## Security Implications

| Risk | Detail |
|---|---|
| Port conflicts | Pod ports compete directly with host ports — binding port 80 blocks all other pods on that node from using port 80 |
| Network isolation bypass | Pod can see all host network interfaces and raw traffic |
| Privilege escalation risk | Combined with `hostPID`, can monitor or interfere with all host processes |
| Broader attack surface | If the container is compromised, attacker has direct host network access |

## Legitimate Use Cases

| Use Case | Why hostNetwork Is Needed |
|---|---|
| **CNI plugins** (Calico, Flannel, Cilium) | Must configure host routing rules and virtual bridges |
| **Node-level monitoring** (Prometheus node-exporter, Datadog agent) | Must read actual host network metrics (eth0 bandwidth, packet drops) |
| **Network troubleshooting tools** | Need raw access to host interfaces |
| **High-performance workloads** | Eliminates overlay network overhead (rare, specialized) |

## Notes
- `hostNetwork: true` should be avoided for application workloads
- CNI DaemonSet pods (like `kube-flannel`) use this by design
- Requires no special port mapping — pod listens on host ports directly
- Best paired with `hostPID: true` only for monitoring agents (see Q61 DaemonSet)
