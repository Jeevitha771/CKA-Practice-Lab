# Q108. Configure CoreDNS to Forward `example.com` to External DNS 8.8.8.8

## Task
Configure CoreDNS so all DNS queries for `example.com` are forwarded to `8.8.8.8`.
Edit ConfigMap, add forward plugin, reload, and verify.

---

## Phase 1: View Current CoreDNS ConfigMap

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

---

## Phase 2: Add a Zone-Specific Forward Block

```bash
kubectl edit configmap coredns -n kube-system
```

Add a dedicated server block for `example.com` **before** the default `.:53` block:

```
# New block — forward example.com queries to Google DNS
example.com:53 {
    errors
    forward . 8.8.8.8
    cache 30
}

# Existing default block — leave unchanged
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

Save and exit.

> **How it works:** CoreDNS processes server blocks in order. The `example.com:53` block
> matches queries ending in `.example.com` first and forwards them to `8.8.8.8`.
> All other queries fall through to the `.:53` block as before.

---

## Phase 3: Reload CoreDNS

```bash
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system
```

---

## Phase 4: Verify

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it -- sh
```

Inside the shell:

```sh
# Should resolve via 8.8.8.8
nslookup api.example.com
nslookup www.example.com

# Internal cluster DNS should still work
nslookup kubernetes.default.svc.cluster.local
```

Expected for `nslookup api.example.com`:
```
Server:    10.96.0.10           # CoreDNS
Name:      api.example.com
Address 1: <resolved IP>        # Forwarded to 8.8.8.8 which resolved it
```

---

## Forward Plugin Options

```
forward . 8.8.8.8               # Single upstream
forward . 8.8.8.8 8.8.4.4       # Multiple upstreams (failover)
forward . /etc/resolv.conf      # Use node's DNS servers
forward . tls://8.8.8.8 {       # DNS over TLS
  tls_servername dns.google
}
```

---

## Summary: Quick Reference

```bash
# Edit CoreDNS config
kubectl edit configmap coredns -n kube-system
# Add: example.com:53 { forward . 8.8.8.8 }

# Reload
kubectl rollout restart deployment coredns -n kube-system

# Test
kubectl run test --image=busybox:1.28 --rm -it -- nslookup api.example.com
```

---

## Cleanup (Restore Original)

```bash
kubectl edit configmap coredns -n kube-system
# Remove the example.com:53 { ... } block
kubectl rollout restart deployment coredns -n kube-system
```
