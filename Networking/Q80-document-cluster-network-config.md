# Q80. Document Cluster Network Configuration

## Task
Document cluster network configuration:
- CNI plugin in use
- Pod CIDR
- Service CIDR
- Cluster DNS IP
- CNI config files location

## Solution

### 1. Find the CNI Plugin in Use

```bash
kubectl get pods -n kube-system
# Look for: calico, flannel, cilium, weave, canal
```

### 2. Find the Pod CIDR

```bash
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr
# --cluster-cidr=10.244.0.0/16

# Or via kubectl
kubectl cluster-info dump | grep -m1 cluster-cidr
```

### 3. Find the Service CIDR

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
# --service-cluster-ip-range=10.96.0.0/12

# Or via kubectl
kubectl cluster-info dump | grep -m1 service-cluster-ip-range
```

### 4. Find the Cluster DNS IP

```bash
kubectl get svc kube-dns -n kube-system
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP
```

### 5. Find the CNI Config Files Location

```bash
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist   # filename varies by plugin
# e.g. 10-calico.conflist, 10-flannel.conflist, 05-cilium.conf
```

## Complete Network Config Summary Table

| Component | Command | Typical Value |
|---|---|---|
| CNI Plugin | `kubectl get pods -n kube-system` | calico / flannel / cilium |
| Pod CIDR | `grep cluster-cidr kube-controller-manager.yaml` | `10.244.0.0/16` |
| Service CIDR | `grep service-cluster-ip-range kube-apiserver.yaml` | `10.96.0.0/12` |
| Cluster DNS IP | `kubectl get svc kube-dns -n kube-system` | `10.96.0.10` |
| CNI config dir | `ls /etc/cni/net.d/` | `/etc/cni/net.d/` |

## Key Concepts

| Term | Description |
|---|---|
| **CNI Plugin** | Software implementing the CNI spec — assigns IPs, configures routes for pods |
| **Pod CIDR** | IP subnet exclusively for pods; each node gets a slice of this range |
| **Service CIDR** | Virtual IPs for Services (ClusterIPs); intercepted by kube-proxy via iptables/IPVS |
| **Cluster DNS IP** | Static IP of CoreDNS; injected into every pod's `/etc/resolv.conf` automatically |
| **CNI config dir** | `/etc/cni/net.d/` — kubelet reads this on startup to find which CNI binary to use |

## Notes
- Pod CIDR and Service CIDR **must not overlap** with each other or the host network
- The Cluster DNS IP is always inside the Service CIDR range
- CoreDNS runs as a Deployment in `kube-system` and is exposed via the `kube-dns` Service
