# Q83. Verify IP Forwarding Enabled and Bridge Netfilter Module Loaded — Fix Issues

## Task
Verify IP forwarding is enabled on nodes and the `br_netfilter` module is loaded. Fix any issues.

## Why These Are Required for Kubernetes

| Requirement | Why |
|---|---|
| `net.ipv4.ip_forward = 1` | Allows the Linux kernel to route packets between network interfaces (pod ↔ node ↔ other nodes) |
| `br_netfilter` module | Allows iptables to see bridged traffic — required for kube-proxy to work correctly with CNI |

## Check and Fix IP Forwarding

```bash
# Check current value
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1
# Bad:      net.ipv4.ip_forward = 0

# Fix temporarily (lost on reboot)
sysctl -w net.ipv4.ip_forward=1

# Fix permanently
cat <<EOF >> /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply immediately without reboot
sysctl --system

# Verify
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1 ✅
```

## Check and Fix br_netfilter Module

```bash
# Check if module is loaded
lsmod | grep br_netfilter
# If empty → module is NOT loaded

# Load it immediately
modprobe br_netfilter

# Verify it's loaded
lsmod | grep br_netfilter
# br_netfilter           28672  0

# Make it load automatically on every boot
cat <<EOF >> /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

## Full Pre-flight Fix (Apply Both at Once)

```bash
# 1. Load modules
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# 2. Set kernel parameters
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# 3. Verify all
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
lsmod | grep br_netfilter
lsmod | grep overlay
```

## Where These Checks Live in Kubernetes Docs

These are standard **pre-flight requirements** before installing kubeadm. Running `kubeadm init` or joining a node will fail if these are missing.

## Notes
- These settings must be applied on **every node** (control plane + workers)
- `/etc/sysctl.d/k8s.conf` → persists across reboots
- `/etc/modules-load.d/k8s.conf` → loads kernel modules on boot
- `sysctl --system` → applies all `.conf` files in `/etc/sysctl.d/` immediately
