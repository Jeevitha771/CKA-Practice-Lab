# Kubernetes Networking — Linux Fundamentals & Architecture Deep Dive

> Comprehensive reference covering Linux networking primitives, Pod networking internals, CNI, Services, kube-proxy, DNS, and security — written for CKA exam preparation.

---

## Section 1: Linux Container Networking Foundations

### The Pause Container — The Anchor of Pod Networking

Every Kubernetes Pod contains a hidden infrastructure container called the **pause container**. Understanding it is the key to understanding Pod networking.

**The Problem it Solves:**
- A Linux container is an isolated process. If a container dies, its network namespace (its private IP stack) dies with it.
- Kubernetes pods can crash and restart, but the Pod's IP address **must not change**.

**How the Pause Container Works:**
```
Pod
├── pause container  ← always sleeping, never does app logic
│     └── owns the Linux network namespace (IP stack)
├── app-container-1  ← shares pause's network namespace
└── app-container-2  ← shares pause's network namespace
```

- The pause container runs in `sleep` mode forever — it is an **anchor** that keeps the network stack alive
- Application containers are **attached to** the network environment created by pause
- If `app-container-1` crashes and restarts, the pause container stays alive → **IP address is preserved**
- All containers in the same pod share `localhost` — they communicate via `127.0.0.1`

**Why No Port Conflicts Across Pods:**
Each Pod has its **own copy** of the Linux IP network stack (its own network namespace).
- Pod A can run nginx on port 80 → `10.244.0.1:80`
- Pod B can run nginx on port 80 → `10.244.0.2:80`
- Zero conflict — they are in completely separate network namespaces

```bash
# Inspect pause container on a node
crictl ps | grep pause
crictl inspect <pause-container-id>
```

---

### veth Pairs — The Wormhole Between Pod and Node

**The Problem:** The Pod lives in its own isolated network namespace. How does a packet from outside reach it?

**The Solution: `veth` (Virtual Ethernet) pair**

```
Pod Network Namespace          Host Root Network Namespace
┌──────────────────┐           ┌───────────────────────────┐
│  eth0 (veth end) │◄─────────►│ vethXXXXX (other veth end)│
│  10.244.0.5      │  virtual  │        (no IP)             │
│                  │   pipe    │           │                 │
└──────────────────┘           │      Linux Bridge (cni0)   │
                               │           │                 │
                               │     Routing Table           │
                               └───────────────────────────-─┘
```

- **One end** of the veth pair is plugged into the Pod's network namespace as `eth0`
- **Other end** is plugged into the **host machine's root network namespace**
- Once on the host, a Linux **bridge** (`cni0` or `cbr0`) connects all veth ends together
- The host's **routing table** decides where to forward the packet next

> **Analogy:** A veth pair is like a wormhole — enter from the pod's universe, exit into the node's universe.

---

### Pod CIDR — Each Node Gets Its Own Subnet

Every node is assigned a **Pod CIDR** — a dedicated IP range for all pods on that node.

```
Worker Node 1:  Pod CIDR = 10.244.0.0/24   (254 pod IPs)
Worker Node 2:  Pod CIDR = 10.244.1.0/24   (254 pod IPs)
Worker Node 3:  Pod CIDR = 10.244.2.0/24   (254 pod IPs)

Total Cluster Pod CIDR: 10.244.0.0/16      (65,534 pod IPs)
```

The **host routing table** on each node knows: *"Packets for 10.244.0.x → send to veth of pod on this node."*

**Cross-node routing:**
If a packet must go to a pod on another node:
1. It hits the host routing table
2. No local route matches → sent to the **default gateway**
3. Every node acts as a **router** to other nodes
4. The CNI plugin sets all this up automatically

---

## Section 2: CNI — The Automation Hero

### What CNI Does

**CNI (Container Network Interface)** is activated every time a pod is created. It:
1. Creates the veth pair
2. Assigns an IP from the node's Pod CIDR
3. Sets up routing rules
4. Connects the pod to the network

```bash
# CNI config files — kubelet reads these on startup
ls /etc/cni/net.d/
# 10-flannel.conflist    (Flannel)
# 10-calico.conflist     (Calico)
# 05-cilium.conf         (Cilium)

# Check running CNI pods
kubectl get pods -n kube-system | grep -E 'calico|flannel|cilium|weave'
```

### CNI Implementations Compared

| CNI | Mechanism | How it Works | Pros | Cons |
|---|---|---|---|---|
| **Flannel** | Overlay (VXLAN) | Wraps pod packet inside a node-to-node packet (encapsulation) | Works on any network | CPU overhead from encapsulation |
| **Calico** | Direct routing (BGP) | No encapsulation — packet travels as-is | Very fast | Routers must know pod CIDRs |
| **Cilium** | eBPF | Processes packets inside the kernel at near-hardware speed | Blazing fast, deep visibility | Requires modern kernel |

**Overlay Network (Flannel VXLAN) — The Matryoshka Doll:**
```
Original:  [Pod A IP → Pod B IP] [payload]
Wrapped:   [Node A IP → Node B IP] [original packet inside]
```
Node B unpacks the outer packet and delivers the inner one to Pod B.

**eBPF (Cilium) — The Future:**
Instead of maintaining tables of iptables rules, eBPF injects tiny programs **directly into the Linux kernel**. Packets are processed at kernel speed without going through the full networking stack.

---

## Section 3: IP Addressing Architecture

### Why Each Component Knows Different CIDRs

```
kube-apiserver    → knows Service CIDR  (--service-cluster-ip-range)
                    Because: it assigns ClusterIPs to Services on creation

kube-controller-manager → knows Pod CIDR (--cluster-cidr)
                    Because: node-controller carves out /24 subnets for each new node
```

**Verification Commands:**
```bash
# Pod CIDR
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr
# --cluster-cidr=10.244.0.0/16

# Service CIDR
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
# --service-cluster-ip-range=10.96.0.0/12

# Per-node Pod CIDR assignments
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

### CIDR Summary Table

| Network | CIDR Example | Purpose | Must Not Overlap With |
|---|---|---|---|
| Pod CIDR | `10.244.0.0/16` | Pod IP addresses | Service CIDR, Host network |
| Service CIDR | `10.96.0.0/12` | ClusterIP Virtual IPs | Pod CIDR, Host network |
| Node network | `192.168.1.0/24` | Physical node IPs | Pod CIDR, Service CIDR |

---

## Section 4: Services and kube-proxy

### The Problem: Pod IPs Are Ephemeral

Pods die and are replaced constantly. Their IPs change. You can't hardcode a Pod IP in your application.

**Solution: Service (ClusterIP)**
- A Service gets a **stable Virtual IP (VIP)** from the Service CIDR
- The VIP never changes as long as the Service exists
- Pods come and go — the Service VIP stays the same

```
App Pod → Service VIP (10.96.5.50) → kube-proxy → Backend Pod (10.244.2.99)
```

### kube-proxy — The Rule Writer

`kube-proxy` runs on every node. It watches the API server and **writes iptables/IPVS rules** every time Services or Pods change.

```
Service created → kube-proxy writes rule:
  "Packets to 10.96.5.50 → redirect to one of: 10.244.2.99, 10.244.1.87, 10.244.3.11"
```

> **Key insight:** kube-proxy is NOT in the data path. It just writes rules. Traffic flows through the Linux kernel directly — kube-proxy never touches actual packets.

### iptables vs IPVS vs eBPF

| Mode | Data Structure | Lookup Speed | Scales To |
|---|---|---|---|
| **iptables** | Sequential list | O(n) — reads every rule per packet | ~1,000 services |
| **IPVS** | Hash table | O(1) — direct lookup | 10,000+ services |
| **eBPF** (Cilium) | Kernel programs | Kernel-native speed | Massive scale |

> **iptables Analogy:** Finding a name by reading every page of a phone book from page 1.
> **IPVS Analogy:** Jumping directly to the right page using the index.

### Connection Tracking (conntrack)

When kube-proxy does DNAT (translates VIP → real Pod IP), how does the reply find its way back?

**conntrack** = the network's memory:

```
1. Pod A → Service VIP 10.96.5.50
2. iptables DNAT → changes destination to Pod C: 10.244.2.99
3. conntrack records: "Pod A's request was translated to Pod C"
4. Pod C replies to Pod A
5. conntrack intercepts reply → changes source back to 10.96.5.50
6. Pod A sees a clean reply from 10.96.5.50 ✅
```

> **Analogy — The Coat Check:** You hand your coat to the attendant (iptables). They hang it on rack #42 (Pod C) and give you a ticket (conntrack record). When you leave, you show the ticket — attendant fetches the exact coat without you ever knowing which rack it was on.

---

## Section 5: CoreDNS and the Cluster DNS IP

### The "10th IP" — Why DNS Gets a Fixed Address

The Cluster DNS IP (e.g., `10.96.0.10`) is:
- A standard **Service VIP** drawn from the Service CIDR pool
- By convention, Kubernetes reserves the **10th IP** in the Service CIDR block
- Assigned to the `kube-dns` Service in `kube-system`
- **Hardcoded by kubelet** into every Pod's `/etc/resolv.conf`

```bash
# The DNS Service
kubectl get svc kube-dns -n kube-system
# kube-dns   ClusterIP   10.96.0.10   ...

# What a Pod's resolv.conf looks like (injected by kubelet)
kubectl exec <any-pod> -- cat /etc/resolv.conf
# search default.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
```

### Full DNS Resolution Flow: `curl my-database`

```
Pod runs: curl my-database
    │
    ▼
/etc/resolv.conf:
  search default.svc.cluster.local    ← "my-database" expands to full FQDN
  nameserver 10.96.0.10               ← send DNS query here
    │
    ▼
DNS query packet → 10.96.0.10 (VIP — doesn't physically exist)
    │
    ▼
iptables DNAT → real CoreDNS Pod IP (10.244.1.5)
    │
    ▼
CoreDNS answers: my-database.default.svc.cluster.local → 10.96.5.50
    │
    ▼
App sends HTTP to 10.96.5.50 (Database Service VIP)
    │
    ▼
iptables DNAT → real Database Pod (10.244.2.99)
    │
    ▼
Database Pod receives request ✅
```

### DNS Naming Convention

```
<service-name>.<namespace>.svc.cluster.local

Examples:
  backend.default.svc.cluster.local
  database.production.svc.cluster.local
  redis.cache.svc.cluster.local

Short names (within same namespace):
  curl backend          → resolves to backend.default.svc.cluster.local
  curl database         → resolves to database.default.svc.cluster.local
```

> **Benefit of DNS over IPs:** Copy the same app architecture to a different namespace — all DNS names still resolve correctly. No IP hardcoding.

### Reserved Cluster IPs

```
10.96.0.1   → kubernetes (API server Service)
10.96.0.10  → kube-dns (CoreDNS Service)
```

---

## Section 6: Network Security — The Flat Network Problem

### Default Kubernetes Network: Completely Open

By default, **any pod can communicate with any other pod** across all namespaces — no firewalls, no restrictions.

```
Pod A (frontend)  ──► Pod B (backend)   ✅ allowed by default
Pod A (frontend)  ──► Pod C (database)  ✅ allowed by default  ← DANGEROUS
Hacked container  ──► Pod C (database)  ✅ allowed by default  ← NIGHTMARE
```

This is called a **flat network**. Great for development speed. Terrible for security.

**The Risk — Lateral Movement:**
If an attacker compromises one container, they can reach every other service in the cluster — including private databases, internal APIs, and secrets.

### NetworkPolicy — The Firewall Layer

NetworkPolicy objects allow you to define **ingress and egress rules** per namespace/pod selector.

```yaml
# Example: Deny all ingress to production namespace by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}        # applies to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

```yaml
# Then explicitly allow only what's needed
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
```

> **Note:** NetworkPolicy requires a CNI that supports it (Calico, Cilium, Weave). Flannel alone does NOT enforce NetworkPolicy.

---

## Section 7: Quick Reference — Networking Commands

```bash
# === CNI ===
ls /etc/cni/net.d/                                        # CNI config files
kubectl get pods -n kube-system | grep -E 'calico|flannel|cilium|weave'

# === IP Addressing ===
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# === DNS ===
kubectl get svc kube-dns -n kube-system
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- nslookup <service-name>
kubectl exec <pod> -- nslookup <service>.<namespace>.svc.cluster.local

# === Services / kube-proxy ===
kubectl get svc --all-namespaces
kubectl get endpoints <service-name>
iptables-save | grep <service-cluster-ip>
ipvsadm -Ln                                               # if using IPVS mode

# === Pod Connectivity ===
kubectl exec -it <pod> -- ping <other-pod-ip>
kubectl exec -it <pod> -- curl <service-name>

# === Node Networking ===
sysctl net.ipv4.ip_forward                               # must be 1
lsmod | grep br_netfilter                                 # must be loaded
ip route                                                  # routing table
ip link                                                   # network interfaces
ip addr                                                   # IP addresses

# === veth pairs (on node) ===
ip link show type veth
brctl show                                                # bridge interfaces

# === conntrack ===
conntrack -L                                              # list tracked connections
conntrack -L | grep <pod-ip>
```

---

## Section 8: Architecture Summary Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
│                                                                   │
│  ┌──────────────┐          ┌──────────────┐                      │
│  │   Worker 1   │          │   Worker 2   │                      │
│  │              │          │              │                      │
│  │ ┌──────────┐ │          │ ┌──────────┐ │                      │
│  │ │  Pod A   │ │          │ │  Pod C   │ │                      │
│  │ │ pause    │ │          │ │ pause    │ │                      │
│  │ │ 10.244.0.5│ │         │ │10.244.1.6│ │                      │
│  │ └────┬─────┘ │          │ └────┬─────┘ │                      │
│  │      │ veth  │          │      │ veth  │                      │
│  │ ┌────┴─────┐ │          │ ┌────┴─────┐ │                      │
│  │ │  Bridge  │ │          │ │  Bridge  │ │                      │
│  │ │  (cni0)  │ │          │ │  (cni0)  │ │                      │
│  │ └────┬─────┘ │          │ └────┬─────┘ │                      │
│  │      │       │          │      │       │                      │
│  │ ┌────┴──────────────────────────┴─────┐ │                     │
│  │ │          Routing Table + CNI        │ │                     │
│  │ │  10.244.1.0/24 → Node 2 gateway    │ │                     │
│  │ └─────────────────────────────────────┘ │                     │
│  └──────────────┘          └──────────────┘                      │
│                                                                   │
│  kube-proxy (every node): writes iptables / IPVS rules           │
│  CoreDNS (kube-system):   answers DNS queries via 10.96.0.10     │
│  CNI plugin (every node): manages veth, bridges, routing         │
└─────────────────────────────────────────────────────────────────┘
```
