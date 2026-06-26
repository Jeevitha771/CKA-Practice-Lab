# Advanced Kubernetes Networking — Deep Dive Concepts

> Reference guide for CKA exam networking theory, troubleshooting patterns, and internal mechanics.

---

## Section 1: Multi-Homed Nodes and Multiple Interfaces

### The Problem
A multi-homed node has multiple NICs (e.g., `eth0` = public management, `eth1` = private backend). The kubelet must know **which IP to register** with the API server, otherwise pod traffic may route over the wrong interface or fail due to firewalls.

### The Fix — `--node-ip` Flag
```bash
# /var/lib/kubelet/config.yaml  OR  systemd unit
# Add the flag:
--node-ip=<BACKEND_INTERFACE_IP>

# Example: force kubelet to use eth1 (10.0.0.10) instead of eth0 (192.168.1.10)
--node-ip=10.0.0.10
```

### Verify Which IP the Node Registered With
```bash
kubectl get node <node-name> -o wide
# INTERNAL-IP should match the interface you intended
```

> **Analogy — The Two Doors Restaurant:** A restaurant has a fancy front door (management network) and a loading dock in the back (data network). Without `--node-ip`, delivery trucks arrive at the front door. The flag is the sign that says *"All deliveries to the back alley."*

---

## Section 2: CKA Networking Troubleshooting Checklist

### 1. Identify the CNI Plugin

```bash
# Pods stuck in ContainerCreating? CNI may be missing or broken.
ls /etc/cni/net.d/
# Look for: 10-flannel.conflist, 10-calico.conflist, 05-cilium.conf

kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'
# All pods should be Running 1/1
```

---

### 2. Verify IP Forwarding (`net.ipv4.ip_forward`)

```bash
# Check
sysctl net.ipv4.ip_forward
# Must return: net.ipv4.ip_forward = 1

# Fix — temporarily
sysctl -w net.ipv4.ip_forward=1

# Fix — permanently
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/k8s.conf
sysctl --system
```

#### Deep Dive: What Breaks When `ip_forward = 0`?

When `ip_forward=0`, the Linux kernel drops any packet whose destination is **not** the node's own IP.

**How it breaks Services:**
1. Pod A sends a packet to Service VIP (`10.96.0.10`)
2. iptables translates the destination to Pod B's real IP (`10.244.2.5`)
3. Pod B is on a **different node** — the packet must be forwarded out
4. `ip_forward=0` → kernel says *"That's not my IP"* → **packet dropped**
5. Service routing silently fails

> **Analogy — The Post Office Strike:** `ip_forward=1` = post office forwards mail to other cities. `ip_forward=0` = post office on strike — they only accept mail addressed to the post office building itself.

---

### 3. Verify `br_netfilter` Module

```bash
# Check
lsmod | grep br_netfilter
# If empty → module NOT loaded

# Fix — immediately
modprobe br_netfilter

# Fix — persist across reboots
echo "br_netfilter" >> /etc/modules-load.d/k8s.conf

# Also set related sysctl
cat <<EOF >> /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system
```

#### Deep Dive: Why `br_netfilter` Is Critical

Linux bridges (`cni0`) operate at **Layer 2** (MAC addresses). iptables operates at **Layer 3/4** (IP addresses). Without `br_netfilter`, traffic crossing the bridge **never reaches iptables**.

**What breaks without it:**
- Pod A sends a packet to Service VIP (`10.96.0.10`)
- Packet hits the `cni0` bridge (Layer 2)
- Bridge says *"just forward it"* — bypasses iptables entirely
- kube-proxy's DNAT rules are **never applied**
- Packet leaves the node still addressed to `10.96.0.10` — a VIP that doesn't physically exist → **dropped**

**What `br_netfilter` does:**
Forces bridge traffic *up* to Layer 3 (iptables) before it crosses, so Service VIP translation works.

> **Analogy — The Toll Booth Bypass:**
> - Bridge = highway bypass around the city
> - iptables = toll booth in the city center
> - Without `br_netfilter`: cars take the bypass and skip the toll booth (iptables never sees traffic)
> - With `br_netfilter`: a police officer forces every car off the bypass to go through the toll booth first

---

### 4. Debug Pod-to-Pod Connectivity

```bash
# Step 1: Exec into source pod
kubectl exec -it pod-a -- sh

# Step 2: Ping destination pod by IP
ping <POD-B-IP>

# Step 3: If ping fails — check node routing table
exit
ip route
# Should see a route: 10.244.x.0/24 via <other-node-ip>

# Step 4: Check iptables
iptables-save | grep -i DROP
iptables-save | grep <POD-B-IP>

# Step 5: Test DNS separately
kubectl exec -it pod-a -- nslookup <service-name>
```

**Decision tree:**

```
Pod can't reach another pod
        |
        ├── Ping by IP fails?      → CNI issue: check ip route, br_netfilter, CNI pod logs
        ├── Ping works, DNS fails? → CoreDNS issue: check CoreDNS pods, nslookup
        └── Only cross-node fails? → Routing issue: ip_forward, ip route on nodes
```

---

### 5. Host Networking (`hostNetwork: true`)

```yaml
spec:
  hostNetwork: true   # Pod shares node's exact IP — bypasses CNI overlay
```

- Pod gets **node's IP** instead of a Pod CIDR IP
- Use case: Ingress controllers, network monitoring agents that need to listen on node ports (80/443) without NAT

---

## Section 3: Overlay Networks vs. Direct Routing

### Overlay Networks (e.g., Flannel VXLAN)

| Aspect | Detail |
|---|---|
| How it works | **Encapsulation** — original packet (Pod A→Pod B) wrapped in a new packet (Node A→Node B) |
| Pros | Works on any network; physical routers don't need to know Pod IPs |
| Cons | Slower — packing/unpacking consumes CPU; reduces MTU |

> **Analogy — The Matryoshka Doll:** The inner envelope says "Pod A → Pod B." Node A stuffs it inside a FedEx box saying "Node A → Node B." Node B opens the box, reads the inner envelope, delivers to Pod B.

---

### Direct Routing (e.g., Calico BGP, AWS VPC CNI)

| Aspect | Detail |
|---|---|
| How it works | **No encapsulation** — packet travels directly as-is (Pod A→Pod B) |
| Pros | Extremely fast; no CPU overhead |
| Cons | Physical routers must be programmed (via BGP) to know which node hosts which Pod CIDR |

---

## Section 4: Kubernetes Services Internals

### Myth-Busting: kube-proxy Is NOT a Proxy

`kube-proxy` does **not** sit in the data path. It is a **Rule Manager** — it watches the API server and writes `iptables` / IPVS rules into the Linux kernel. All traffic flows directly through the kernel, never through the `kube-proxy` pod.

---

### iptables vs. IPVS vs. eBPF

| Mode | Mechanism | Analogy | Use Case |
|---|---|---|---|
| **iptables** (default) | Sequential list of rules — checked one by one per packet | Reading a phonebook from page 1 every time | Small clusters (<1000 services) |
| **IPVS** | Hash table — O(1) lookup per packet | Using the index at the back of the book | Large clusters (>1000 services) |
| **eBPF** (Cilium) | Custom programs injected into the deepest kernel layer — bypasses iptables/IPVS | A dedicated express lane that skips all checkpoints | High-performance / modern clusters |

---

### DNAT and Connection Tracking (conntrack)

How a Service VIP actually routes traffic:

```
1. Pod A sends packet → destination: Service VIP (10.96.0.10)
2. iptables DNAT rule fires → replaces 10.96.0.10 with real Pod IP (10.244.2.5)
3. conntrack records: "Pod A's request was translated from VIP → Pod C"
4. Pod C receives request, sends reply to Pod A
5. Kernel intercepts reply, checks conntrack → changes sender back to 10.96.0.10
6. Pod A sees reply from 10.96.0.10 — consistent, transparent
```

> **Analogy — The Coat Check:** You hand your coat to the attendant (iptables), who hangs it on rack #42 (backend Pod) and gives you a ticket (conntrack record). When you leave, hand the ticket back — the attendant fetches the exact coat without you ever knowing which rack it was on.

---

## Section 5: DNS Flow Deep Dive — How `curl my-database` Works

### The Journey of a Single DNS Request

```
Frontend Pod runs: curl http://my-database
```

**Step 1 — App asks Linux for help**
App doesn't know what IP `my-database` is. It calls the OS.

**Step 2 — `/etc/resolv.conf` (injected by kubelet)**
```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
```
- `search` → short name `my-database` expands to `my-database.default.svc.cluster.local`
- `nameserver` → "ask 10.96.0.10 for directions" (this is the kube-dns Service VIP)

**Step 3 — DNS packet intercepted by iptables**
Pod sends DNS query to `10.96.0.10` (VIP — doesn't physically exist).
iptables DNAT rule → translates to real CoreDNS Pod IP (`10.244.1.5`).
CoreDNS Pod receives the query.

**Step 4 — CoreDNS answers**
CoreDNS watches the Kubernetes API. It answers:
`my-database.default.svc.cluster.local → 10.96.5.50` (the Database Service VIP).

**Step 5 — App sends real HTTP traffic**
App now sends HTTP GET to `10.96.5.50`.

**Step 6 — Second iptables intercept**
iptables DNAT rule → translates `10.96.5.50` to real Database Pod IP (`10.244.2.99`).
Packet delivered to the Database Pod. 

```
Frontend Pod
     │
     │  curl my-database
     ▼
/etc/resolv.conf ──► send DNS to 10.96.0.10 (kube-dns VIP)
     │
     ▼
iptables DNAT ──► real CoreDNS pod (10.244.1.5)
     │
     ▼
CoreDNS returns ──► 10.96.5.50 (Database Service VIP)
     │
     ▼
iptables DNAT ──► real Database Pod (10.244.2.99)
     │
     ▼
Database Pod receives request ✅
```

> **The "10th IP" Summary:** `10.96.0.10` is a permanently stable virtual front door to the CoreDNS phonebook. Without it, pods would have no way to find the constantly-changing ephemeral IPs of other applications in the cluster.

---

## Quick CKA Troubleshooting Reference

```bash
# CNI plugin check
ls /etc/cni/net.d/
kubectl get pods -n kube-system | grep -E 'calico|flannel|cilium|weave'

# IP forwarding
sysctl net.ipv4.ip_forward              # Must be 1
sysctl -w net.ipv4.ip_forward=1        # Fix

# br_netfilter
lsmod | grep br_netfilter               # Must show output
modprobe br_netfilter                   # Fix

# Pod connectivity test
kubectl exec -it <source-pod> -- ping <dest-pod-ip>
kubectl exec -it <source-pod> -- nslookup <service-name>

# Node routing table
ip route

# DNS config in pod
kubectl exec -it <pod> -- cat /etc/resolv.conf

# CoreDNS health
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system <coredns-pod>
```
