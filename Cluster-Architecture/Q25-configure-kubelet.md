# Q25. Configure kubelet on Worker Node

## Task
Configure kubelet on worker node with:
- Cluster DNS: `10.96.0.10`
- Cluster domain: `cluster.local`
- Max pods: `150`

---

## Method 1: kubelet Configuration File (Recommended)

```bash
# Edit or create the kubelet config file
vi /var/lib/kubelet/config.yaml
```

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDNS:
- "10.96.0.10"
clusterDomain: "cluster.local"
maxPods: 150
```

```bash
# Restart kubelet to apply
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

---

## Method 2: kubelet Environment File (Older Clusters)

```bash
# Edit the kubelet environment flags
vi /etc/default/kubelet
# or on RHEL:
vi /etc/sysconfig/kubelet
```

Add:
```
KUBELET_EXTRA_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local --max-pods=150
```

```bash
systemctl daemon-reload
systemctl restart kubelet
```

---

## Verify Configuration

```bash
# Check kubelet is using the settings
ps aux | grep kubelet | grep -E 'cluster-dns|max-pods'

# Or check the resolved config
kubectl get node <node-name> -o yaml | grep -A 5 capacity
# capacity:
#   pods: "150"

# Check DNS settings in a test pod
kubectl run test --image=busybox:1.28 --rm -it -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
```

---

## Kubelet Config File Location

```bash
# Find the config file used by kubelet
systemctl cat kubelet | grep config
# --config=/var/lib/kubelet/config.yaml

# View full resolved kubelet config
kubectl get --raw /api/v1/nodes/<node-name>/proxy/configz | python3 -m json.tool
```

---

## Key kubelet Settings Reference

| Setting | Config Key | Flag |
|---|---|---|
| Cluster DNS IP | `clusterDNS` | `--cluster-dns` |
| Cluster domain | `clusterDomain` | `--cluster-domain` |
| Max pods | `maxPods` | `--max-pods` |
| Cgroup driver | `cgroupDriver` | `--cgroup-driver` |
| Image GC thresholds | `imageGCHighThresholdPercent` | `--image-gc-high-threshold` |
