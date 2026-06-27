# Q109. Scale CoreDNS to 3 Replicas with Pod Anti-Affinity

## Task
Scale CoreDNS to 3 replicas for high availability.
Configure pod anti-affinity so each CoreDNS pod runs on a different node.

---

## Phase 1: Check Current CoreDNS State

```bash
kubectl get deployment coredns -n kube-system
```

```
NAME      READY   UP-TO-DATE   AVAILABLE   DESIRED
coredns   2/2     2            2           2
```

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

Note the current node distribution.

---

## Phase 2: Scale to 3 Replicas

```bash
kubectl scale deployment coredns --replicas=3 -n kube-system
```

Verify:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## Phase 3: Add Pod Anti-Affinity

Edit the CoreDNS deployment to ensure pods spread across different nodes:

```bash
kubectl edit deployment coredns -n kube-system
```

Add the `affinity` section inside `spec.template.spec`:

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          # Prefer different nodes — soft requirement (won't block scheduling if impossible)
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: k8s-app
                  operator: In
                  values:
                  - kube-dns
              topologyKey: kubernetes.io/hostname
```

Save and exit — Kubernetes will rolling-restart the CoreDNS pods.

> **`preferredDuring...` vs `requiredDuring...`:**
> - `preferred` = soft rule — CoreDNS can still start if only 1 node exists
> - `required` = hard rule — pods WON'T schedule if they can't spread (risky for DNS)
>
> Use `preferred` for CoreDNS so DNS always stays available even in single-node clusters.

---

## Phase 4: Verify Node Distribution

```bash
kubectl rollout status deployment coredns -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

Expected (3-node cluster):
```
NAME                READY   STATUS    NODE
coredns-xxx-aaa     1/1     Running   controlplane
coredns-xxx-bbb     1/1     Running   worker-1
coredns-xxx-ccc     1/1     Running   worker-2
```

> Each pod is on a different node — DNS remains available even if one node goes down.

---

## Phase 5: Verify DNS Still Works

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
```

---

## Why HA for CoreDNS Matters

```
Node-1 crashes → coredns pod on Node-1 dies
                  ↓
Node-2 and Node-3 still have coredns pods
                  ↓
All pods in cluster continue resolving DNS ✅
```

Without anti-affinity, all 3 CoreDNS pods could land on the same node — one node failure
would take out all DNS.

---

## Summary: Quick Reference

```bash
# Scale
kubectl scale deployment coredns --replicas=3 -n kube-system

# Add anti-affinity
kubectl edit deployment coredns -n kube-system
# Add: spec.template.spec.affinity.podAntiAffinity (preferred, topologyKey: hostname)

# Verify
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```
