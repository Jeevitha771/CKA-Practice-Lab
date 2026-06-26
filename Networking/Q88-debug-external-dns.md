# Q88. Debug: Pod Cannot Resolve External DNS — Internal Works, External Fails

## Task
Pod cannot resolve `google.com` but can resolve internal Kubernetes services.  
Debug and fix the CoreDNS configuration.

---

## Phase 1: Break the Environment (Lab Setup)

```bash
# 1. Deploy a test pod
kubectl run test-dns --image=busybox:1.28 -- sleep 3600

# 2. Edit the CoreDNS ConfigMap to point the upstream forwarder at a fake/dead IP
kubectl edit configmap coredns -n kube-system
```

Inside the editor, find:
```
forward . /etc/resolv.conf
```
Change it to:
```
forward . 198.51.100.1
```
Save and exit (`Esc` → `:wq` → `Enter` in vim).

```bash
# 3. Restart CoreDNS to apply the broken config
kubectl rollout restart deployment coredns -n kube-system
```

---

## Phase 2: Verify the Symptoms

### Internal DNS — Works ✅

```bash
kubectl exec -it test-dns -- nslookup kubernetes.default.svc.cluster.local
```

```
Server:    10.96.0.10
Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1
```

CoreDNS handles internal cluster names itself (no upstream needed) — so this works fine.

### External DNS — Fails ❌

```bash
kubectl exec -it test-dns -- nslookup google.com
```

```
;; connection timed out; no servers could be reached
```

The request is forwarded upstream to `198.51.100.1`, which is unreachable.

---

## Phase 3: Debugging Steps

### Step 1: Check CoreDNS Logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Look for errors like:
```
[ERROR] plugin/errors: 2 google.com. SERVFAIL
[ERROR] plugin/forward: no upstream found for google.com: i/o timeout
```

This confirms CoreDNS is alive but cannot reach its upstream forwarder.

### Step 2: Inspect the CoreDNS Corefile

```bash
kubectl describe configmap coredns -n kube-system
```

Look for the `.:53` block:
```
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
    }
    forward . 198.51.100.1     # <-- Root cause: dead upstream IP
    cache 30
    loop
    reload
    loadbalance
}
```

**Root Cause:** The `forward .` directive tells CoreDNS where to send all non-cluster DNS queries.  
It is pointing at `198.51.100.1` — a documentation IP that is unreachable.

---

## Phase 4: Fix the Configuration

### Option A — Use the Node's DNS (recommended)

```bash
kubectl edit configmap coredns -n kube-system
```

Change:
```
forward . 198.51.100.1
```
To:
```
forward . /etc/resolv.conf
```

This makes CoreDNS forward external queries using the same DNS servers the underlying node uses.

### Option B — Hardcode a Public DNS Server

```bash
kubectl edit configmap coredns -n kube-system
```

Change:
```
forward . 198.51.100.1
```
To:
```
forward . 8.8.8.8
```

Use this when the node's own DNS is unreliable.

---

### Apply the Fix

ConfigMap changes are **not** automatically applied to running pods. Restart CoreDNS:

```bash
kubectl rollout restart deployment coredns -n kube-system

# Wait for rollout to complete
kubectl rollout status deployment coredns -n kube-system
```

---

## Phase 5: Verify the Fix

```bash
# Test external DNS again
kubectl exec -it test-dns -- nslookup google.com
```

```
Server:    10.96.0.10
Name:      google.com
Address 1: 142.250.x.x
Address 2: 142.250.x.x
```

✅ External DNS resolution is restored.

---

## Summary: Internal Works, External Fails — Checklist

| Check | Command |
|---|---|
| CoreDNS pods running? | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| CoreDNS error logs? | `kubectl logs -n kube-system -l k8s-app=kube-dns` |
| Corefile `forward .` setting? | `kubectl describe configmap coredns -n kube-system` |
| Internal DNS resolves? | `kubectl exec -it <pod> -- nslookup kubernetes.default` |
| External DNS resolves? | `kubectl exec -it <pod> -- nslookup google.com` |

---

## Key Concept: The Corefile `forward` Directive

| Setting | Meaning |
|---|---|
| `forward . /etc/resolv.conf` | Use the host node's configured DNS servers |
| `forward . 8.8.8.8` | Use Google's public DNS directly |
| `forward . 198.51.100.1` | ❌ Points to a dead IP — external DNS will fail |

---

## Cleanup

```bash
kubectl delete pod test-dns
```
