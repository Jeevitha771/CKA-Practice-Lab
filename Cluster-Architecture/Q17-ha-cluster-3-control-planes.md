# Q17. Configure HA Cluster with 3 Control Plane Nodes

## Task
Configure an HA cluster with:
- 3 control plane nodes: `192.168.1.101`, `192.168.1.102`, `192.168.1.103`
- Load balancer at `192.168.1.50:6443`
- Pod network CIDR: `10.244.0.0/16`

---

## Architecture Overview

```
                    ┌──────────────────────────┐
                    │  Load Balancer (HAProxy)  │
                    │     192.168.1.50:6443     │
                    └───────────┬──────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │ control-plane-1│   │ control-plane-2│   │ control-plane-3│
  │ 192.168.1.101  │   │ 192.168.1.102  │   │ 192.168.1.103  │
  └───────────────┘   └───────────────┘   └───────────────┘
```

---

## Phase 1: Configure Load Balancer (HAProxy)

```bash
# On the load balancer node (192.168.1.50)
apt-get install -y haproxy

cat >> /etc/haproxy/haproxy.cfg <<EOF
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-connect
    option tcplog
    server control-plane-1 192.168.1.101:6443 check
    server control-plane-2 192.168.1.102:6443 check
    server control-plane-3 192.168.1.103:6443 check
EOF

systemctl restart haproxy
systemctl enable haproxy
```

---

## Phase 2: Initialize First Control Plane Node

```bash
# On 192.168.1.101 (first control plane)
kubeadm init \
  --control-plane-endpoint "192.168.1.50:6443" \
  --upload-certs \
  --pod-network-cidr 10.244.0.0/16 \
  --apiserver-advertise-address 192.168.1.101
```

Save the output — it contains two join commands:
1. For additional **control plane** nodes (includes `--control-plane --certificate-key`)
2. For **worker** nodes

---

## Phase 3: Configure kubectl on First Control Plane

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Install CNI (Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

## Phase 4: Join Remaining Control Plane Nodes

```bash
# On 192.168.1.102 and 192.168.1.103 (use exact output from init step)
kubeadm join 192.168.1.50:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key> \
  --apiserver-advertise-address <this-node-ip>
```

---

## Phase 5: Verify HA Control Plane

```bash
kubectl get nodes
# NAME               STATUS   ROLES           AGE
# control-plane-1    Ready    control-plane   10m
# control-plane-2    Ready    control-plane   5m
# control-plane-3    Ready    control-plane   3m

# Check etcd cluster health
kubectl exec -n kube-system etcd-control-plane-1 -- \
  etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

---

## Key HA Concepts

| Concept | Detail |
|---|---|
| `--control-plane-endpoint` | Must point to load balancer — stable for all nodes |
| `--upload-certs` | Uploads control plane certs to cluster secret (24h TTL) |
| `--certificate-key` | Used by joining control plane nodes to download certs |
| etcd quorum | 3 nodes = can tolerate 1 failure; 5 nodes = 2 failures |
