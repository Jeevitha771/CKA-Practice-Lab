# Q106. Modify CoreDNS ConfigMap — Custom DNS Entries

## Task
Add custom DNS overrides to CoreDNS:
- `database.local` → `192.168.1.100`
- `cache.local` → `192.168.1.101`

Reload CoreDNS and verify resolution.

---

## Phase 1: View the Current CoreDNS ConfigMap

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Default Corefile:
```
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

---

## Phase 2: Add Custom DNS Entries Using the `hosts` Plugin

```bash
kubectl edit configmap coredns -n kube-system
```

Add a `hosts` block **above** the `kubernetes` plugin (plugin order matters in CoreDNS):

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready

    # Custom DNS overrides — added for Q106
    hosts {
      192.168.1.100 database.local
      192.168.1.101 cache.local
      fallthrough
    }

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

Save and exit (`:wq` in vim).

> **Why `fallthrough`?** It tells the `hosts` plugin to pass the query to the next
> plugin if no match is found — so normal cluster DNS keeps working.

---

## Phase 3: Reload CoreDNS

CoreDNS has a `reload` plugin that auto-detects ConfigMap changes every 30s.
To apply immediately, restart the deployment:

```bash
kubectl rollout restart deployment coredns -n kube-system

# Wait for rollout
kubectl rollout status deployment coredns -n kube-system
```

---

## Phase 4: Verify the Custom DNS Entries

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it -- sh
```

Inside the shell:

```sh
# Resolve custom entries
nslookup database.local
nslookup cache.local

# Verify cluster DNS still works
nslookup kubernetes.default.svc.cluster.local
```

Expected:
```
# database.local
Server:    10.96.0.10
Name:      database.local
Address 1: 192.168.1.100

# cache.local
Server:    10.96.0.10
Name:      cache.local
Address 1: 192.168.1.101

# cluster DNS still works
Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1
```

---

## Alternative: Using `rewrite` Plugin for Name Override

If you want to redirect a name to an existing service instead of a bare IP:

```
rewrite name database.local my-database-service.default.svc.cluster.local
```

This is useful when migrating applications that use hostnames like `database.local`.

---

## Summary: Quick Reference

```bash
# Edit CoreDNS config
kubectl edit configmap coredns -n kube-system

# Restart to apply immediately
kubectl rollout restart deployment coredns -n kube-system

# Test from a pod
kubectl run test --image=busybox:1.28 --rm -it -- nslookup database.local
```

---

## Cleanup (Restore Original ConfigMap)

```bash
kubectl edit configmap coredns -n kube-system
# Remove the hosts { ... } block you added
kubectl rollout restart deployment coredns -n kube-system
```
