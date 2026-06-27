# Q117. Default Deny-All NetworkPolicy Then Specific Allow Policies

## Task
1. Create a default deny-all NetworkPolicy for a namespace
2. Create specific allow policies for required traffic

---

## Phase 1: Set Up the Environment

```bash
kubectl create namespace secured

kubectl run frontend --image=nginx  -n secured --labels="app=frontend"
kubectl run backend  --image=nginx  -n secured --labels="app=backend"
kubectl run database --image=nginx  -n secured --labels="app=database"
```

---

## Phase 2: Apply Default Deny-All (Ingress + Egress)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secured
spec:
  podSelector: {}           # all pods
  policyTypes:
  - Ingress
  - Egress
  # No ingress: or egress: rules = deny everything
EOF
```

At this point, **no pod can communicate** — not even DNS.

---

## Phase 3: Add Required Allow Policies

### Allow 1 — DNS for all pods (always needed first)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: secured
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
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

### Allow 2 — Frontend can receive traffic on port 80

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: secured
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
    # No `from:` = allow from any source on this port
EOF
```

### Allow 3 — Frontend can reach backend on port 8080

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: secured
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

### Allow 4 — Backend can reach database on port 5432

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: secured
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
EOF
```

---

## Phase 4: Verify All Policies

```bash
kubectl get networkpolicy -n secured
```

```
NAME                       POD-SELECTOR    AGE
default-deny-all           <none>          2m
allow-dns                  <none>          90s
allow-frontend-ingress     app=frontend    60s
allow-frontend-to-backend  app=backend     40s
allow-backend-to-db        app=database    20s
```

---

## Traffic Matrix After Policies

| Traffic | Allowed? |
|---|---|
| Internet → frontend:80 | ✅ |
| frontend → backend:8080 | ✅ |
| backend → database:5432 | ✅ |
| frontend → database:5432 | ❌ |
| All pods → kube-dns:53 | ✅ |
| All pods → internet | ❌ |

---

## The Correct Pattern for Securing a Namespace

```
Step 1: default-deny-all     → lock everything down
Step 2: allow-dns            → restore DNS first
Step 3: allow-<specific>     → open only what's needed
```

> Never skip Step 2. A namespace with default-deny but no DNS allow will cause
> all pod communication to silently fail at the DNS lookup stage.

---

## Summary: Quick Reference

```bash
# 1. Deny everything
kubectl apply -f default-deny-all.yaml

# 2. Allow DNS (always)
kubectl apply -f allow-dns.yaml

# 3. Allow specific traffic
kubectl apply -f allow-specific.yaml

# Verify
kubectl get networkpolicy -n secured
```

---

## Cleanup

```bash
kubectl delete namespace secured
```
