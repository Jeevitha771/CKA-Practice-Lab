# Q112. NetworkPolicy: Deny All Ingress + Allow Frontend + Allow Monitoring

## Task
In namespace `production`:
1. Deny all ingress by default
2. Allow ingress from pods with label `app=frontend` on port `8080`
3. Allow ingress from namespace `monitoring` on port `9090`

---

## Phase 1: Set Up the Environment

```bash
kubectl create namespace production
kubectl create namespace monitoring

# Backend pod in production
kubectl run backend --image=nginx -n production --labels="app=backend"

# Frontend pod (in default ns, with label app=frontend)
kubectl run frontend --image=busybox:1.28 --labels="app=frontend" -- sleep 3600

# Monitoring pod in monitoring namespace
kubectl run prometheus --image=busybox:1.28 -n monitoring -- sleep 3600
```

---

## Phase 2: Apply the NetworkPolicies

### Policy 1 — Deny All Ingress (default deny)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}         # applies to ALL pods in namespace
  policyTypes:
  - Ingress               # deny all ingress; no ingress rules = deny all
EOF
```

> `podSelector: {}` = select all pods.
> `policyTypes: [Ingress]` with no `ingress:` rules = **deny all ingress**.

---

### Policy 2 — Allow frontend pods on port 8080

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend          # applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # only from pods with app=frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

---

### Policy 3 — Allow monitoring namespace on port 9090

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring   # from monitoring namespace
    ports:
    - protocol: TCP
      port: 9090
EOF
```

> Label the monitoring namespace (required for `namespaceSelector`):
> ```bash
> kubectl label namespace monitoring kubernetes.io/metadata.name=monitoring
> ```
> Note: In Kubernetes 1.21+, this label is automatically set on all namespaces.

---

## Phase 3: Verify

```bash
kubectl get networkpolicy -n production
```

```
NAME                    POD-SELECTOR    AGE
default-deny-ingress    <none>          30s
allow-frontend          app=backend     30s
allow-monitoring        app=backend     30s
```

---

## Phase 4: Test

### Test 1 — Frontend pod → backend:8080 (should ALLOW)

```bash
BACKEND_IP=$(kubectl get pod backend -n production -o jsonpath='{.status.podIP}')

kubectl exec -it frontend -- wget -qO- --timeout=3 $BACKEND_IP:8080
# Expected: 200 OK (nginx welcome page)
```

### Test 2 — Monitoring pod → backend:9090 (should ALLOW)

```bash
kubectl exec -it prometheus -n monitoring -- wget -qO- --timeout=3 $BACKEND_IP:9090
# Expected: Connection refused (nginx doesn't serve 9090) but NOT timeout/dropped
# A refused connection means the network policy ALLOWED it — the port just isn't open
```

### Test 3 — Default pod → backend:80 (should DENY)

```bash
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- --timeout=3 $BACKEND_IP:80
# Expected: timeout — packet dropped by NetworkPolicy
```

---

## How Multiple Policies Work Together

NetworkPolicies are **additive** — they combine with OR logic:
- Policy 1 denies everything (baseline)
- Policy 2 adds: allow frontend→8080
- Policy 3 adds: allow monitoring→9090

A packet is allowed if **any** policy permits it.

---

## Summary: Quick Reference

```bash
# Apply all 3 policies
kubectl apply -f default-deny-ingress.yaml
kubectl apply -f allow-frontend.yaml
kubectl apply -f allow-monitoring.yaml

# Verify
kubectl get networkpolicy -n production
kubectl describe networkpolicy allow-frontend -n production

# Test
kubectl exec -it frontend -- wget -qO- --timeout=3 <backend-ip>:8080
```

---

## Cleanup

```bash
kubectl delete namespace production monitoring
```
