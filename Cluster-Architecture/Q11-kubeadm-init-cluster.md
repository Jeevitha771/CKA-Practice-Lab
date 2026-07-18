# Q11. Initialize Kubernetes Cluster with kubeadm

## Task
Initialize a Kubernetes cluster using kubeadm with:
- Pod network CIDR: `10.244.0.0/16`
- Kubernetes version: v1.28.0
- Control plane endpoint: `k8s-master.local`
- Configure kubectl for admin user

---

## Phase 1: Pre-flight — Install Prerequisites

```bash
# Disable swap (required by kubelet)
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Set sysctl parameters
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# Install containerd, kubeadm, kubelet, kubectl (Ubuntu/Debian)
apt-get update && apt-get install -y apt-transport-https curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet=1.28.0-* kubeadm=1.28.0-* kubectl=1.28.0-*
apt-mark hold kubelet kubeadm kubectl
```

---

## Phase 2: Initialize the Cluster

```bash
kubeadm init \
  --kubernetes-version=v1.28.0 \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint=k8s-master.local \
  --apiserver-advertise-address=<control-plane-ip>
```

Expected output (save the join command at the bottom):
```
Your Kubernetes control-plane has initialized successfully!
...
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-master.local:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Phase 3: Configure kubectl for Admin User

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
# NAME           STATUS     ROLES           AGE
# k8s-master     NotReady   control-plane   1m
# (NotReady until CNI is installed)
```

---

## Phase 4: Install Flannel CNI (for pod CIDR 10.244.0.0/16)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Wait for node to become Ready
kubectl get nodes -w
# NAME           STATUS   ROLES           AGE
# k8s-master     Ready    control-plane   3m
```

---

## Verify Full Cluster Health

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
```

```
Kubernetes control plane is running at https://k8s-master.local:6443
CoreDNS is running at https://k8s-master.local:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

---

## Key kubeadm init Flags

| Flag | Purpose |
|---|---|
| `--pod-network-cidr` | CIDR for pod IPs — must match CNI plugin |
| `--control-plane-endpoint` | Stable hostname/IP for HA or DNS |
| `--kubernetes-version` | Pin specific version |
| `--apiserver-advertise-address` | IP the API server binds to |
| `--service-cidr` | CIDR for service IPs (default: `10.96.0.0/12`) |
