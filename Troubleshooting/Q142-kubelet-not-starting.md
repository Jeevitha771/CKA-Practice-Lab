# Q142. Kubelet Not Starting on a Node

## Task
Kubelet not starting on a node. Check systemd logs, kubelet configuration, and certificate issues.

---

## Phase 1: Check kubelet Status

```bash
# On the node (SSH in)
systemctl status kubelet
```

Common failure states:
```
● kubelet.service - kubelet: The Kubernetes Node Agent
     Active: failed (Result: exit-code)
    Process: ExecStart=/usr/bin/kubelet ... (code=exited, status=1/FAILURE)
```

---

## Phase 2: Read kubelet Logs

```bash
# Last 50 lines
journalctl -u kubelet -n 50 --no-pager

# Follow live
journalctl -u kubelet -f
```

---

## Phase 3: Common Errors and Fixes

### Error 1: Certificate Expired or Missing

```
Error: "x509: certificate has expired or is not yet valid"
Error: "failed to run Kubelet: unable to load bootstrap kubeconfig"
```

```bash
# Check certificate expiry
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
# notAfter=Jan 15 00:00:00 2024 GMT  ← expired

# Renew certificates (from control plane)
kubeadm alpha certs renew all
# or
kubeadm certs renew all
```

---

### Error 2: Config File Error

```
Error: "failed to load config file \"/var/lib/kubelet/config.yaml\""
Error: "unknown field \"invalidField\""
```

```bash
# Check syntax
cat /var/lib/kubelet/config.yaml

# Validate it's valid YAML
python3 -c "import yaml; yaml.safe_load(open('/var/lib/kubelet/config.yaml'))"
```

---

### Error 3: Wrong API Server Address

```
Error: "Unable to register node ... connection refused"
Error: "dial tcp 10.0.0.1:6443: connect: no route to host"
```

```bash
# Check the bootstrap kubeconfig
cat /etc/kubernetes/kubelet.conf | grep server
# server: https://192.168.1.100:6443  ← should be API server IP

# Verify API server is reachable
curl -k https://192.168.1.100:6443/healthz
# ok
```

---

### Error 4: cgroup Driver Mismatch

```
Error: "failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: \"cgroupfs\" is different from docker cgroup driver: \"systemd\""
```

```bash
# Check containerd cgroup driver
grep SystemdCgroup /etc/containerd/config.toml
# SystemdCgroup = true

# Check kubelet cgroup driver
grep cgroupDriver /var/lib/kubelet/config.yaml
# cgroupDriver: systemd

# They must match. Update kubelet config:
sed -i 's/cgroupDriver: cgroupfs/cgroupDriver: systemd/' /var/lib/kubelet/config.yaml
systemctl restart kubelet
```

---

### Error 5: Swap Still Enabled

```
Error: "failed to run Kubelet: running with swap on is not supported"
```

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
systemctl restart kubelet
```

---

## Phase 4: Restart and Verify

```bash
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
# Active: active (running) ✅

# On control plane — node should become Ready
kubectl get nodes
```

---

## Debug Checklist

| Check | Command |
|---|---|
| kubelet status | `systemctl status kubelet` |
| Recent logs | `journalctl -u kubelet -n 50` |
| Certificate validity | `openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates` |
| API server reachable | `curl -k https://<api-server>:6443/healthz` |
| Config file valid | `cat /var/lib/kubelet/config.yaml` |
| cgroup driver match | compare kubelet config vs containerd config |
| Swap disabled | `swapon --show` |
