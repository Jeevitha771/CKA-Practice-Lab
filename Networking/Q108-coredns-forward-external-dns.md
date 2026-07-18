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
# New block ΓÇö forward example.com queries to Google DNS
example.com:53 {
    errors
    forward . 8.8.8.8
    cache 30
}

# Existing default block ΓÇö leave unchanged
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

---

In CoreDNS, every single word inside that Server Block is a **plugin**. CoreDNS is built entirely out of plugins that execute from top to bottom.

When you teach this to your students, you can explain `errors` and `cache 30` as the **logging** and **performance** layers for that specific block of traffic.

Here is exactly what each one does.

## `errors` (The Logging Plugin)

The `errors` plugin does exactly what it sounds like: if anything goes wrong while processing a query inside this Server Block, it prints the error message to the standard output (stdout) of the CoreDNS container.

**Why this matters for your students:**
In the previous troubleshooting lab, we used the command `kubectl -n kube-system logs -l k8s-app=kube-dns` to read the CoreDNS logs.

If you do **not** include the `errors` word in your Server Block, CoreDNS will fail silently. If `8.8.8.8` goes offline, or if there is a network timeout, the application trying to reach `example.com` will fail, but when you check the CoreDNS logs, they will be completely empty. Adding `errors` ensures that failures are visible for debugging.

## `cache 30` (The Performance Plugin)

The `cache` plugin creates an in-memory storage layer for DNS answers, and the `30` represents the Time-To-Live (TTL) in **seconds**.

**How it works in practice:**

1. A pod asks for `api.example.com`.
2. CoreDNS realizes it doesn't know the answer, so it forwards the request to Google (`8.8.8.8`).
3. Google replies: *"api.example.com is at 93.184.215.14."*
4. CoreDNS hands that IP back to the pod, but because of `cache 30`, it also **memorizes** that answer for the next 30 seconds.

If a hundred other pods ask for `api.example.com` during those next 30 seconds, CoreDNS will answer them instantly from its own RAM. It will not send another request out to `8.8.8.8` until the 30 seconds expire.

**Why this matters for your students:**
In a real-world Kubernetes cluster, thousands of pods might be asking for the same database or external API simultaneously. Without the `cache` plugin, CoreDNS would blast the external network with thousands of identical DNS queries, causing high latency, wasted bandwidth, and potentially getting your cluster rate-limited or blocked by external DNS providers.
