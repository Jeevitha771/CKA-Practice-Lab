# Q113. NetworkPolicy: Allow Same-Namespace Egress + kube-dns + Deny All Other Egress

## Task
Create a NetworkPolicy that:
1. Allows egress to pods **in the same namespace**
2. Allows egress to `kube-dns` on port `53`
3. Denies all other egress

---

## Phase 1: Set Up the Environment

```bash
kubectl create namespace app-ns
kubectl run app-pod  --image=busybox:1.28 -n app-ns --labels="app=myapp" -- sleep 3600
kubectl run peer-pod --image=nginx         -n app-ns --labels="app=peer"
kubectl run external --image=nginx         -n default --labels="app=external"
```

---

## Phase 2: Apply the NetworkPolicy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: app-ns
spec:
  podSelector: {}         # applies to ALL pods in app-ns
  policyTypes:
  - Egress
  egress:
  # Rule 1: Allow egress to pods in the SAME namespace
  - to:
    - podSelector: {}     # empty = all pods in same namespace
  # Rule 2: Allow egress to kube-dns (port 53 UDP and TCP)
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
```

> ⚠️ **Always allow port 53 (kube-dns)** in egress policies.
> Without it, the pod cannot resolve any DNS names — even internal cluster services.

---

## Phase 3: Verify

```bash
kubectl get networkpolicy -n app-ns
```

```
NAME              POD-SELECTOR   AGE
restrict-egress   <none>         20s
```

---

## Phase 4: Test

### Test 1 — DNS resolution (should ALLOW)

```bash
kubectl exec -n app-ns app-pod -- nslookup kubernetes.default
# Expected: resolves successfully
```

### Test 2 — Same namespace pod (should ALLOW)

```bash
PEER_IP=$(kubectl get pod peer-pod -n app-ns -o jsonpath='{.status.podIP}')
kubectl exec -n app-ns app-pod -- wget -qO- --timeout=3 $PEER_IP:80
# Expected: 200 OK
```

### Test 3 — Pod in different namespace (should DENY)

```bash
EXT_IP=$(kubectl get pod external -n default -o jsonpath='{.status.podIP}')
kubectl exec -n app-ns app-pod -- wget -qO- --timeout=3 $EXT_IP:80
# Expected: timeout — egress to different namespace is denied
```

### Test 4 — External internet (should DENY)

```bash
kubectl exec -n app-ns app-pod -- wget -qO- --timeout=3 http://google.com
# Expected: timeout — external egress is denied
```

---

## Common Mistake: Forgetting DNS in Egress Policies

Without the kube-dns rule, even internal service names fail:
```
nslookup kubernetes.default → connection timed out
```

Because the DNS query to port 53 is blocked by the egress policy.
**Always add the DNS egress rule whenever you restrict egress.**

---

## Summary: Quick Reference

```bash
# Egress policy structure
policyTypes: [Egress]
egress:
- to: [{podSelector: {}}]                         # same namespace
- to: [{namespaceSelector: ..., podSelector: ...}] # kube-dns
  ports: [{port: 53, protocol: UDP}, {port: 53, protocol: TCP}]
```

---

## Cleanup

```bash
kubectl delete namespace app-ns
kubectl delete pod external -n default
```
