# Q86. Debug: Pod A in `frontend` Cannot Reach Pod B in `backend`

## Task
Debug cross-namespace connectivity failure: check NetworkPolicies, DNS resolution, test with curl, check endpoints.

---

## Phase 1: Setup the Broken Environment

```bash
# 1. Create namespaces
kubectl create ns frontend
kubectl create ns backend

# 2. Deploy frontend pod
kubectl run pod-a -n frontend --image=nginx

# 3. Deploy backend pod with label app=backend-app
kubectl run pod-b -n backend --image=nginx --labels="app=backend-app"

# 4. Create a Service with an INTENTIONALLY WRONG selector (app=backend-wrong)
kubectl create service clusterip service-b -n backend --tcp=80:80
kubectl set selector svc service-b 'app=backend-wrong' -n backend

# 5. Apply a default-deny NetworkPolicy on the backend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-all
  namespace: backend
spec:
  podSelector: {}   # Selects ALL pods in backend namespace
  policyTypes:
  - Ingress
EOF
```

---

## Phase 2: Diagnose Bug #1 — Misconfigured Service (Connection Refused)

```bash
# 1. Test connection — fails instantly with "Connection refused" or "Could not resolve"
kubectl exec -it pod-a -n frontend -- curl -m 3 http://service-b.backend.svc.cluster.local

# 2. Check the Endpoints — shows NO backend IPs attached
kubectl get endpoints service-b -n backend
# NAME        ENDPOINTS   AGE
# service-b   <none>      ...   <-- Empty! No pods matched the selector

# 3. Fix the Service selector to match the actual pod label
kubectl set selector svc service-b 'app=backend-app' -n backend

# 4. Verify Endpoints now has an IP
kubectl get endpoints service-b -n backend
# NAME        ENDPOINTS         AGE
# service-b   10.244.x.x:80     ...   <-- Pod IP appears!
```

**Root Cause:** The Service selector `app=backend-wrong` matched zero pods, so the Service had no Endpoints and could not route traffic.

---

## Phase 3: Diagnose Bug #2 — NetworkPolicy Blocking Traffic (Connection Timed Out)

```bash
# 1. Test again — this time it HANGS, then fails with "Connection timed out"
kubectl exec -it pod-a -n frontend -- curl -m 3 http://service-b.backend.svc.cluster.local

# 2. Check if any NetworkPolicies exist in backend namespace
kubectl get networkpolicies -n backend
# NAME        POD-SELECTOR   AGE
# block-all   <none>         ...   <-- This is blocking ALL ingress!

# 3. Describe it to understand the rule
kubectl describe networkpolicy block-all -n backend
```

### Fix: Label the Frontend Namespace and Allow Ingress

```bash
# 1. Add a label to the frontend namespace (used in the NetworkPolicy selector)
kubectl label namespace frontend team=frontend-team

# 2. Apply an allow policy targeting traffic from frontend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: frontend-team
EOF

# 3. Test again — should now succeed
kubectl exec -it pod-a -n frontend -- curl -m 3 http://service-b.backend.svc.cluster.local
# Expected: Welcome to nginx!
```

---

## Debugging Decision Tree

```
Pod A cannot reach Pod B (cross-namespace)
          |
          ├── curl returns "Connection refused" instantly?
          │       └── Check Endpoints: kubectl get endpoints <svc> -n <ns>
          │           If <none>: Service selector is wrong — fix with kubectl set selector
          │
          ├── curl hangs and times out?
          │       └── Check NetworkPolicies: kubectl get networkpolicies -n <ns>
          │           If a default-deny exists: add an allow NetworkPolicy
          │
          └── curl returns "Could not resolve host"?
                  └── DNS issue: check CoreDNS pods and kubectl exec -- nslookup
```

---

## Key Commands Cheatsheet

```bash
# Check service endpoints
kubectl get endpoints <svc-name> -n <namespace>

# Check all network policies in a namespace
kubectl get networkpolicies -n <namespace>

# Check pod labels
kubectl get pod <pod-name> -n <namespace> --show-labels

# Test connectivity directly
kubectl exec -it <pod> -n <namespace> -- curl -m 3 http://<service>.<ns>.svc.cluster.local
```

---

## Cleanup

```bash
kubectl delete ns frontend backend
```
