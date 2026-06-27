# Q107. Pod Cannot Resolve Service Names — Debug CoreDNS

## Task
A pod cannot resolve Kubernetes service names. Debug by checking:
- CoreDNS pod status and logs
- Service existence
- DNS from inside the pod (`nslookup`/`dig`)
- `/etc/resolv.conf`
- CoreDNS ConfigMap

---

## Phase 1: Break the Environment (Lab Setup)

```bash
# Deploy a test pod
kubectl run test-pod --image=busybox:1.28 -- sleep 3600

# Deliberately break CoreDNS by scaling it to 0
kubectl scale deployment coredns --replicas=0 -n kube-system
```

---

## Phase 2: Observe the Symptom

```bash
kubectl exec -it test-pod -- nslookup kubernetes.default
```

```
nslookup: can't resolve 'kubernetes.default'
```

---

## Phase 3: Debug Step-by-Step

### Step 1 — Check CoreDNS pod status

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                      READY   STATUS    RESTARTS
coredns-xxxxxx-yyy        0/1     Pending   0
```

> ❌ CoreDNS pods are not running — all DNS will fail.

### Step 2 — Check CoreDNS logs (when pods are running)

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Look for:
```
[ERROR] plugin/errors: 2 kubernetes.default. NXDOMAIN
[ERROR] Restart failed
[FATAL] Failed to start server
```

### Step 3 — Verify the service exists

```bash
# Does the service you're trying to resolve actually exist?
kubectl get service kubernetes -n default
kubectl get service <your-service-name> -n <namespace>
```

### Step 4 — Check /etc/resolv.conf inside the pod

```bash
kubectl exec -it test-pod -- cat /etc/resolv.conf
```

Expected (healthy):
```
nameserver 10.96.0.10       # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

> If `nameserver` is missing or points to a wrong IP, the pod's DNS is misconfigured.
> Check the kubelet `--cluster-dns` flag.

```bash
# Verify CoreDNS service ClusterIP (should be 10.96.0.10 or similar)
kubectl get service kube-dns -n kube-system
```

```
NAME       TYPE        CLUSTER-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP   10d
```

### Step 5 — Verify CoreDNS ConfigMap

```bash
kubectl describe configmap coredns -n kube-system
```

Look for syntax errors in the Corefile:
- Missing closing `}` braces
- Broken `forward .` directive
- Invalid plugin names

### Step 6 — Test DNS with dig (if available)

```bash
kubectl run dig-test --image=tutum/dnsutils --rm -it -- dig kubernetes.default.svc.cluster.local
```

Or using nslookup with explicit DNS server:

```bash
kubectl exec -it test-pod -- nslookup kubernetes.default 10.96.0.10
```

---

## Phase 4: Fix the Issue

```bash
# Restore CoreDNS replicas
kubectl scale deployment coredns --replicas=2 -n kube-system

# Wait for pods to start
kubectl rollout status deployment coredns -n kube-system
```

---

## Phase 5: Verify the Fix

```bash
kubectl exec -it test-pod -- nslookup kubernetes.default
```

```
Server:    10.96.0.10
Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1
```

✅ DNS resolution restored.

---

## Debug Checklist

| Check | Command | What to Look For |
|---|---|---|
| CoreDNS pods running | `kubectl get pods -n kube-system -l k8s-app=kube-dns` | `Running 1/1` |
| CoreDNS logs | `kubectl logs -n kube-system -l k8s-app=kube-dns` | No ERROR lines |
| CoreDNS service IP | `kubectl get svc kube-dns -n kube-system` | ClusterIP matches pod's resolv.conf |
| Pod resolv.conf | `kubectl exec <pod> -- cat /etc/resolv.conf` | nameserver = CoreDNS ClusterIP |
| Service exists | `kubectl get svc <name> -n <namespace>` | Not NotFound |
| CoreDNS ConfigMap | `kubectl describe cm coredns -n kube-system` | Valid Corefile syntax |
| Network policy | `kubectl get networkpolicy -A` | Not blocking port 53 |

---

## Cleanup

```bash
kubectl delete pod test-pod
```
