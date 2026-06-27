# Q114. NetworkPolicy: Three-Tier App — Frontend → Backend → Database

## Task
Implement NetworkPolicies for a three-tier architecture:
- `frontend` can access `backend`
- `backend` can access `database`
- `database` accepts **only** from `backend`
- All tiers can access DNS (port 53)

---

## Phase 1: Set Up the Environment

```bash
kubectl create namespace three-tier

kubectl run frontend --image=nginx  -n three-tier --labels="tier=frontend"
kubectl run backend  --image=nginx  -n three-tier --labels="tier=backend"
kubectl run database --image=nginx  -n three-tier --labels="tier=database"
```

---

## Phase 2: Apply Default Deny for All Ingress + Egress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: three-tier
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

---

## Phase 3: Allow DNS for All Tiers

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: three-tier
spec:
  podSelector: {}           # all pods
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

---

## Phase 4: Allow Frontend → Backend (port 8080)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: backend             # policy applied TO backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend        # allow FROM frontend pods
    ports:
    - protocol: TCP
      port: 8080
EOF
```

Also allow frontend egress to backend:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

---

## Phase 5: Allow Backend → Database (port 5432)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: database            # policy applied TO database pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend         # allow FROM backend only
    ports:
    - protocol: TCP
      port: 5432
EOF
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
EOF
```

---

## Phase 6: Verify All Policies

```bash
kubectl get networkpolicy -n three-tier
```

```
NAME                        POD-SELECTOR      AGE
default-deny-all            <none>            1m
allow-dns                   <none>            50s
allow-frontend-to-backend   tier=backend      40s
allow-frontend-egress       tier=frontend     30s
allow-backend-to-database   tier=database     20s
allow-backend-egress        tier=backend      10s
```

---

## Traffic Matrix

| Source | Destination | Port | Allowed? |
|---|---|---|---|
| frontend | backend | 8080 | ✅ Yes |
| frontend | database | 5432 | ❌ No |
| backend | database | 5432 | ✅ Yes |
| database | backend | any | ❌ No |
| any | internet | any | ❌ No |
| any | kube-dns | 53 | ✅ Yes |

---

## Architecture Diagram

```
[Internet]  ✗
     ✗
[Frontend] ──8080──→ [Backend] ──5432──→ [Database]
    ↑                    ↑                    ↑
    |                    |                    |
    └────────────────────┴────────────────────┘
                         |
                    kube-dns:53 (all tiers allowed)
```

---

## Cleanup

```bash
kubectl delete namespace three-tier
```
