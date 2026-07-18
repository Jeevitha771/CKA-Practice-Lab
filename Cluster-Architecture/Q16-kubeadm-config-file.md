# Q16. kubeadm Configuration File for Custom Cluster Init

## Task
Create a kubeadm configuration file for initializing a cluster with:
- API server advertise address: `192.168.1.100`
- Pod subnet: `10.244.0.0/16`
- Service subnet: `10.96.0.0/12`

---

## Solution: kubeadm-config.yaml

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.1.100"   # IP the API server listens on
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  taints: []                           # remove control-plane taint to allow workloads (optional)
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "v1.28.0"
controlPlaneEndpoint: "192.168.1.100:6443"
networking:
  podSubnet: "10.244.0.0/16"          # must match CNI plugin CIDR
  serviceSubnet: "10.96.0.0/12"       # ClusterIP range
  dnsDomain: "cluster.local"
etcd:
  local:
    dataDir: /var/lib/etcd
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

---

## Initialize Using the Config File

```bash
kubeadm init --config kubeadm-config.yaml
```

---

## Verify the Configuration Was Applied

```bash
# Check API server advertise address
kubectl get pods -n kube-system kube-apiserver-<node> -o yaml | grep advertise-address

# Check service CIDR
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range
# --service-cluster-ip-range=10.96.0.0/12

# Check pod CIDR
kubectl get nodes -o jsonpath='{.items[0].spec.podCIDR}'
# 10.244.0.0/24 (first node's slice of the CIDR)
```

---

## Print Default Config for Reference

```bash
# Print default InitConfiguration
kubeadm config print init-defaults

# Print default KubeletConfiguration
kubeadm config print init-defaults --component-configs KubeletConfiguration
```

---

## Key Notes

- Config file takes precedence over command-line flags
- `podSubnet` in `networking` must match what the CNI plugin expects
- `controlPlaneEndpoint` should be a load balancer address for HA clusters
- `cgroupDriver: systemd` must match containerd's cgroup driver setting
