# Q84. Configure Pod to Use a Specific Network Interface or IP from a Multi-Homed Node

## Task
Configure a pod to use a specific network interface or IP address from a multi-homed node (a node with multiple network interfaces/IPs).

## Background — What is a Multi-Homed Node?

A multi-homed node has **multiple network interfaces**, for example:
- `eth0` → 192.168.1.10 (management network)
- `eth1` → 10.0.0.10 (storage/data network)
- `eth2` → 172.16.0.10 (external/DMZ network)

By default, pods get an IP from the CNI plugin on the primary interface. To target a specific interface, you need Multus CNI or annotation-based IP selection.

## Method 1: hostNetwork + nodeSelector (Simple — Uses Node's IP Directly)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: specific-network-pod
spec:
  hostNetwork: true           # Uses the node's network stack directly
  nodeSelector:
    kubernetes.io/hostname: node1   # Pin to a specific node

  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f pod.yaml
kubectl get pod specific-network-pod -o wide
# Pod IP = node's IP
```

## Method 2: Multus CNI — Attach Additional Network Interface

Multus is a CNI meta-plugin that enables attaching multiple network interfaces to a single pod.

### Step 1: Define a NetworkAttachmentDefinition

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: data-network
spec:
  config: '{
    "cniVersion": "0.3.1",
    "type": "macvlan",
    "master": "eth1",
    "mode": "bridge",
    "ipam": {
      "type": "static",
      "addresses": [{"address": "10.0.0.100/24", "gateway": "10.0.0.1"}]
    }
  }'
```

### Step 2: Annotate the Pod to Use It

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multus-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: data-network   # Attach eth1-based network
spec:
  containers:
  - name: app
    image: nginx
```

```bash
kubectl apply -f nad.yaml
kubectl apply -f pod.yaml

# Verify pod has multiple interfaces
kubectl exec -it multus-pod -- ip addr
# Should show: eth0 (default CNI), net1 (data-network on eth1)
```

## Method 3: environment variable / downward API (Read Node IP into Pod)

If you just need the pod to **know** which node IP to bind to:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-ip-aware-pod
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: NODE_IP
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP    # Injects the node's primary IP into the container
```

```bash
kubectl apply -f pod.yaml
kubectl exec -it node-ip-aware-pod -- printenv NODE_IP
# 192.168.1.10
```

## Summary

| Method | When to Use |
|---|---|
| `hostNetwork: true` | Pod needs to bind to node ports directly |
| Multus CNI + NetworkAttachmentDefinition | Pod needs dedicated secondary interface (storage, data plane) |
| `status.hostIP` via downwardAPI | App just needs to know the node IP (no interface change) |
| `nodeSelector` | Pin pod to a node with the desired interface |

## Notes
- Multi-homed pod support requires **Multus CNI** installed on the cluster
- `hostNetwork` shares ALL interfaces; Multus gives **dedicated secondary** interfaces
- This is common in telecoms (SR-IOV), high-performance computing, and storage networks
