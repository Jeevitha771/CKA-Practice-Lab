# Q111. Configure CoreDNS Custom Upstream DNS for All External Queries (SKIM)

> **SKIM ΓÇö Overlaps with Q108. Low additional exam value.**
> Know the single-line change needed. Do not deep-dive.

## What to Know

To use a custom upstream DNS server (e.g. corporate DNS `10.0.0.53`) for **all** external queries,
change the `forward .` directive in the `.:53` block:

```bash
kubectl edit configmap coredns -n kube-system
```

Change:
```
forward . /etc/resolv.conf
```
To:
```
forward . 8.8.8.8
```

Or for multiple upstreams with failover:
```
forward . 8.8.8.8 10.0.0.54
```

Reload:
```bash
kubectl rollout restart deployment coredns -n kube-system
```

Verify:
```bash
kubectl run test --image=busybox:1.28 --rm -it -- nslookup google.com
```

> **Difference from Q108:** Q108 forwards only `example.com` to a specific server.
> This changes the upstream for **all** external queries globally.
> For zone-specific forwarding see Q108.
