# Q28–Q30. Cluster Upgrade v1.27 → v1.28

## Q28 Task
Upgrade cluster from v1.27.2 to v1.28.0:
- Check upgrade plan
- Upgrade control plane (kubeadm, kubelet, kubectl)
- Upgrade worker nodes

## Q29 Task
Upgrade kubelet and kubectl on worker node `worker-1` from v1.27.0 to v1.28.0:
- Drain node, upgrade packages, restart services, uncordon.

## Q30 Task
Before cluster upgrade, verify: current version, available upgrades, component health, etcd backup status.

---

## Q30: Pre-Upgrade Verification

```bash
# Current version
kubectl version --short
# Client Version: v1.27.2
# Server Version: v1.27.2

# Available upgrade paths
kubeadm upgrade plan
# COMPONENT                 CURRENT   TARGET
# kube-apiserver            v1.27.2   v1.28.0
# ...

# Component health
kubectl get componentstatuses
kubectl get nodes
kubectl get pods -n kube-system

# Backup etcd before upgrade (critical!)
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-pre-upgrade-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## Q28: Upgrade Control Plane Node

### Step 1: Upgrade kubeadm on Control Plane

```bash
# Unhold, upgrade, rehold
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.28.0-*
apt-mark hold kubeadm

# Verify
kubeadm version
# kubeadm version: v1.28.0
```

### Step 2: Apply the Upgrade

```bash
kubeadm upgrade plan
kubeadm upgrade apply v1.28.0
# [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.28.0".
```

### Step 3: Drain the Control Plane Node

```bash
kubectl drain <control-plane-node> \
  --ignore-daemonsets \
  --delete-emptydir-data
```

### Step 4: Upgrade kubelet and kubectl

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.28.0-* kubectl=1.28.0-*
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
```

### Step 5: Uncordon the Control Plane Node

```bash
kubectl uncordon <control-plane-node>
kubectl get nodes
# NAME              STATUS   VERSION
# control-plane     Ready    v1.28.0
```

---

## Q29: Upgrade Worker Node `worker-1`

```bash
# On CONTROL PLANE: drain the worker node
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# On WORKER-1: upgrade kubeadm, apply node upgrade
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.28.0-*
apt-mark hold kubeadm

kubeadm upgrade node

# On WORKER-1: upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.28.0-* kubectl=1.28.0-*
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

# On CONTROL PLANE: uncordon the worker node
kubectl uncordon worker-1

# Verify
kubectl get nodes
# NAME          STATUS   VERSION
# worker-1      Ready    v1.28.0
```

---

## Upgrade Order (Critical)

```
1. Backup etcd
2. Upgrade control plane kubeadm
3. kubeadm upgrade apply
4. Upgrade control plane kubelet/kubectl
5. For each worker: drain → upgrade kubeadm + kubelet/kubectl → uncordon
```

> ⚠️ **Never skip versions**. Upgrade one minor version at a time (1.27 → 1.28, not 1.26 → 1.28).
