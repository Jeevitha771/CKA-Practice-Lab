# Q26–Q27. Verify System Prerequisites and Configure Cgroup Driver

## Q26 Task
Verify system prerequisites on a node:
- Swap disabled
- Kernel modules loaded (`br_netfilter`, `overlay`)
- sysctl parameters correct
- Container runtime installed

## Q27 Task
Configure kubelet to use `systemd` as cgroup driver. Restart kubelet and verify.

---

## Q26: Verify All Prerequisites

### 1. Check Swap is Disabled

```bash
free -h
# Swap: line should show 0B

swapon --show
# (empty output = swap disabled)
```

If swap is ON:
```bash
swapoff -a
# Persist across reboots
sed -i '/ swap / s/^/#/' /etc/fstab
```

---

### 2. Check Kernel Modules

```bash
lsmod | grep -E 'br_netfilter|overlay'
# br_netfilter           36864  0
# overlay               147456  0
```

If missing:
```bash
modprobe br_netfilter
modprobe overlay

# Persist across reboots
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

---

### 3. Check sysctl Parameters

```bash
sysctl net.bridge.bridge-nf-call-iptables
# net.bridge.bridge-nf-call-iptables = 1

sysctl net.bridge.bridge-nf-call-ip6tables
# net.bridge.bridge-nf-call-ip6tables = 1

sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 1
```

If wrong:
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

---

### 4. Check Container Runtime

```bash
systemctl is-active containerd
# active

containerd --version
# containerd containerd.io 1.7.6

crictl info | grep runtimeType
# "runtimeType": "io.containerd.runc.v2"
```

---

## Q27: Configure Cgroup Driver to `systemd`

### Step 1: Update containerd

```bash
# Ensure containerd uses systemd cgroup driver
grep SystemdCgroup /etc/containerd/config.toml
# Should be: SystemdCgroup = true

# If not present:
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

### Step 2: Update kubelet

```bash
vi /var/lib/kubelet/config.yaml
```

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd     # must match container runtime
```

```bash
systemctl daemon-reload
systemctl restart kubelet
```

---

### Verify cgroup Driver

```bash
# Check kubelet
kubectl get node <node> -o json | python3 -c "
import json, sys
n = json.load(sys.stdin)
print(n['status']['nodeInfo']['osImage'])
"

# Check containerd
crictl info | grep -i cgroup
# "cgroupDriver": "systemd"

# Check kubelet resolved config
systemctl status kubelet
ps aux | grep kubelet | grep cgroupDriver
```

---

## Why cgroup Driver Must Match

> If containerd uses `systemd` but kubelet uses `cgroupfs`, pods will fail to start with errors like:
> `"failed to create containerd task: failed to create shim: cgroup v2: must be a unified hierarchy"`
> Always ensure both use the **same** driver — `systemd` is the modern recommended choice.
