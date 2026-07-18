# Q23. Firewall Ports for Kubernetes Nodes

## Task
Document all required firewall ports for control plane nodes and worker nodes (incoming and bidirectional).

---

## Control Plane Node Ports

| Protocol | Port | Direction | Component | Purpose |
|---|---|---|---|---|
| TCP | 6443 | Inbound | kube-apiserver | Kubernetes API server |
| TCP | 2379-2380 | Inbound | etcd | etcd client/peer API |
| TCP | 10250 | Inbound | kubelet | Kubelet API |
| TCP | 10259 | Inbound | kube-scheduler | kube-scheduler metrics |
| TCP | 10257 | Inbound | kube-controller-manager | controller-manager metrics |

## Worker Node Ports

| Protocol | Port | Direction | Component | Purpose |
|---|---|---|---|---|
| TCP | 10250 | Inbound | kubelet | Kubelet API |
| TCP | 30000-32767 | Inbound | kube-proxy | NodePort services |

---

## Open Ports with iptables (Linux)

```bash
# --- Control Plane Node ---

# API server
iptables -A INPUT -p tcp --dport 6443 -j ACCEPT

# etcd
iptables -A INPUT -p tcp --dport 2379:2380 -j ACCEPT

# kubelet API
iptables -A INPUT -p tcp --dport 10250 -j ACCEPT

# scheduler and controller-manager (metrics/healthz)
iptables -A INPUT -p tcp --dport 10257 -j ACCEPT
iptables -A INPUT -p tcp --dport 10259 -j ACCEPT

# --- Worker Node ---

# kubelet API
iptables -A INPUT -p tcp --dport 10250 -j ACCEPT

# NodePort services
iptables -A INPUT -p tcp --dport 30000:32767 -j ACCEPT
```

---

## Open Ports with firewalld (RHEL/CentOS)

```bash
# Control plane
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10257/tcp
firewall-cmd --permanent --add-port=10259/tcp

# Worker nodes
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp

firewall-cmd --reload
```

---

## CNI Plugin Additional Ports

| Plugin | Extra Ports |
|---|---|
| Flannel | UDP 8285, UDP 8472 (VXLAN) |
| Calico | TCP 179 (BGP), IP proto 4 (IPIP) |
| Weave | TCP 6783, UDP 6783, UDP 6784 |

---

## Verify Open Ports

```bash
ss -tlnp | grep -E '6443|2379|10250|10257|10259'
# or
netstat -tlnp | grep -E '6443|2379|10250'
```
