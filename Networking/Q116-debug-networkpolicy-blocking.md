# Q116. NetworkPolicy Blocking Legitimate Traffic — Debug

## Task
A NetworkPolicy is blocking legitimate traffic. Debug the policy rules and test connectivity.

---

## Phase 1: Break the Environment (Lab Setup)

```bash
kubectl create namespace debug-np

kubectl run server --image=nginx  -n debug-np --labels="app=server,tier=backend"
kubectl run client --image=busybox:1.28 -n debug-np --labels="app=client" -- sleep 3600

# Apply a policy with a WRONG selector — blocks the legitimate client
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client
  namespace: debug-np
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client-wrong      # WRONG label — client has app=client
    ports:
    - protocol: TCP
      port: 80
EOF
```

---

## Phase 2: Observe the Symptom

```bash
SERVER_IP=$(kubectl get pod server -n debug-np -o jsonpath='{.status.podIP}')
kubectl exec -n debug-np client -- wget -qO- --timeout=3 $SERVER_IP:80
```

```
wget: can't connect to remote host: Connection timed out
```

---

## Phase 3: Debug Step-by-Step

### Step 1 — List all NetworkPolicies in the namespace

```bash
kubectl get networkpolicy -n debug-np
```

```
NAME           POD-SELECTOR   AGE
allow-client   app=server     2m
```

### Step 2 — Describe the policy and inspect selectors

```bash
kubectl describe networkpolicy allow-client -n debug-np
```

```
Spec:
  PodSelector: app=server
  Ingress:
  - From:
      PodSelector: app=client-wrong     # ← wrong selector
    Ports: 80/TCP
```

### Step 3 — Check actual pod labels

```bash
kubectl get pods -n debug-np --show-labels
```

```
NAME     READY   LABELS
server   1/1     app=server,tier=backend
client   1/1     app=client              # ← has app=client, NOT app=client-wrong
```

> **Root cause:** Policy allows `app=client-wrong` but client pod has label `app=client`.

### Step 4 — Check if there is a default deny policy

```bash
kubectl get networkpolicy -n debug-np
# Are there any policies with podSelector: {} and no rules? → that's a default deny
```

### Step 5 — Test direct pod access (check if issue is NetworkPolicy or app)

```bash
# Deploy a pod that WILL match the wrong label to confirm policy mechanism works
kubectl run testmatch --image=busybox:1.28 -n debug-np \
  --labels="app=client-wrong" -- sleep 3600

kubectl exec -n debug-np testmatch -- wget -qO- --timeout=3 $SERVER_IP:80
# Expected: 200 OK — confirms policy mechanism works, just wrong label
```

---

## Phase 4: Fix the Policy

```bash
kubectl patch networkpolicy allow-client -n debug-np \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/ingress/0/from/0/podSelector/matchLabels/app","value":"client"}]'
```

Or edit directly:

```bash
kubectl edit networkpolicy allow-client -n debug-np
# Change: app: client-wrong
# To:     app: client
```

---

## Phase 5: Verify the Fix

```bash
kubectl exec -n debug-np client -- wget -qO- --timeout=3 $SERVER_IP:80
```

```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
```

✅ Traffic is now allowed.

---

## NetworkPolicy Debug Checklist

| Check | Command | What to Look For |
|---|---|---|
| List policies | `kubectl get netpol -n <ns>` | All policies in namespace |
| Inspect selectors | `kubectl describe netpol <name> -n <ns>` | `From:` / `To:` selectors |
| Pod labels | `kubectl get pods --show-labels -n <ns>` | Labels match policy selectors |
| Default deny? | `kubectl get netpol -n <ns>` | Any policy with empty podSelector and no rules |
| Namespace labels | `kubectl get namespace --show-labels` | Required for namespaceSelector |
| Test allowed pod | Create pod with matching labels | Confirm policy allows it |
| Test blocked pod | Create pod without matching labels | Confirm policy blocks it |

---

## Common Policy Mistakes

| Mistake | Result | Fix |
|---|---|---|
| Wrong label in `podSelector` | Legitimate pods blocked | Match label exactly |
| Missing `namespaceSelector` | Cross-namespace traffic blocked | Add nsSelector |
| Forgot DNS egress | Pod can't resolve names | Add port 53 egress rule |
| Wrong namespace for policy | Policy in wrong namespace | Check `metadata.namespace` |
| `podSelector: {}` with no rules | Denies everything | Add explicit ingress/egress rules |

---

## Cleanup

```bash
kubectl delete namespace debug-np
```
