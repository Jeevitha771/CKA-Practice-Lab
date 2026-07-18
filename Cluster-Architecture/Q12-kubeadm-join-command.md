# Q12. Generate kubeadm Join Command for a Worker Node

## Task
Generate a kubeadm join command for a worker node. Control plane is at `192.168.1.100:6443`.

---

## Option A: Generate From an Existing Cluster

```bash
# Run on the control plane node
kubeadm token create --print-join-command
```

Output:
```
kubeadm join 192.168.1.100:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:a94ef...
```

---

## Option B: Create a Custom Token First

```bash
# Create a token valid for 2 hours
kubeadm token create --ttl 2h --print-join-command

# List all existing tokens
kubeadm token list
# TOKEN                     TTL   EXPIRES                USAGES   DESCRIPTION
# abcdef.0123456789abcdef   23h   2024-01-16T12:00:00Z   auth     <none>
```

---

## Option C: Manually Construct the Join Command

```bash
# Get the token
kubeadm token list

# Get the CA cert hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | \
  sed 's/^.* //'

# Construct the command
kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Running the Join Command on the Worker Node

```bash
# On the WORKER node (as root):
kubeadm join 192.168.1.100:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:a94ef...
```

---

## Verify Worker Joined

```bash
# On control plane:
kubectl get nodes
# NAME           STATUS   ROLES           AGE
# k8s-master     Ready    control-plane   10m
# worker-1       Ready    <none>          1m
```

---

## Notes

- Default token TTL is **24 hours** — expired tokens must be regenerated
- `--discovery-token-ca-cert-hash` prevents MITM attacks during join
- To join as a **control plane node** (HA), add `--control-plane` and `--certificate-key` flags
