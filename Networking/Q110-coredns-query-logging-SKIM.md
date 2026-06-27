# Q110. Enable CoreDNS Query Logging (SKIM)

> **SKIM — Low CKA exam probability.** Know that the `log` plugin exists and where to add it.
> You do not need to memorise this for the exam.

## What to Know

Add `log` to the CoreDNS Corefile to enable query logging:

```bash
kubectl edit configmap coredns -n kube-system
```

Add `log` inside the `.:53` block:

```
.:53 {
    errors
    log                 # ← add this line to enable query logging
    health
    ready
    kubernetes cluster.local ...
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

Reload:
```bash
kubectl rollout restart deployment coredns -n kube-system
```

View logs:
```bash
kubectl logs -n kube-system -l k8s-app=kube-dns -f
```

Each DNS query now appears:
```
[INFO] 10.244.0.5:12345 - 1234 "A IN kubernetes.default.svc.cluster.local. udp 52 false 512" NOERROR qr,aa,rd 106 0.000123s
```

> ⚠️ Remove `log` in production — high DNS query volume generates massive log output
> and can fill node disk space quickly. Only enable for short debugging sessions.
