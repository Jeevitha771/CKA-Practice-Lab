# Q31–Q32. Rollback Failed Upgrade / Upgrade Control Plane Only

## Q31 Task
Rollback a failed worker node upgrade to previous version. Investigate failure cause.

## Q32 Task
Upgrade only the control plane components without upgrading worker nodes. Verify mixed-version cluster operation.

---

## Q31: Rollback a Failed Worker Node Upgrade

### Step 1: Investigate the Failure

```bash
# Check node status
kubectl get nodes
# NAME       STATUS     VERSION
# worker-1   NotReady   v1.28.0   ← failed upgrade

# Check kubelet on the worker node
ssh worker-1 "systemctl status kubelet"
ssh worker-1 "journalctl -u kubelet -n 50 --no-pager"
```

Common failure messages:
```
# Version skew: kubelet 1.28 cannot communicate with API server 1.27
# Error: "server version v1.27.2 is too old..."

# Config incompatibility
# Error: "failed to load config file /var/lib/kubelet/config.yaml"
```

---

### Step 2: Rollback kubelet on Worker Node

```bash
# On worker-1: downgrade to previous version
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.27.2-* kubectl=1.27.2-*
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
```

---

### Step 3: Verify Recovery

```bash
kubectl get nodes
# NAME       STATUS   VERSION
# worker-1   Ready    v1.27.2

# Uncordon if still cordoned
kubectl uncordon worker-1
```

---

## Q32: Upgrade Only Control Plane (Mixed-Version Cluster)

```bash
# Step 1: Upgrade kubeadm on control plane
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.28.0-*
apt-mark hold kubeadm

# Step 2: Apply control plane upgrade ONLY
kubeadm upgrade apply v1.28.0
# [upgrade/successful] Control plane upgraded to v1.28.0

# Step 3: Upgrade control plane kubelet/kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.28.0-* kubectl=1.28.0-*
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet

# Skip worker node upgrade for now
```

### Verify Mixed-Version Cluster

```bash
kubectl get nodes
# NAME              STATUS   ROLES           VERSION
# control-plane     Ready    control-plane   v1.28.0  ← upgraded
# worker-1          Ready    <none>          v1.27.2  ← still old
# worker-2          Ready    <none>          v1.27.2  ← still old

# Test that workloads still run on worker nodes
kubectl get pods -A -o wide | grep worker
```

---

## Version Skew Policy

| Component | Maximum Skew |
|---|---|
| kubelet vs kube-apiserver | kubelet can be up to **2 minor versions older** |
| kubectl vs kube-apiserver | ±1 minor version |
| kube-proxy vs kube-apiserver | Same minor version |

> ✅ v1.28 API server + v1.27 kubelet workers = **supported** (within 2 version skew)
> ❌ v1.28 API server + v1.25 kubelet = **not supported**
