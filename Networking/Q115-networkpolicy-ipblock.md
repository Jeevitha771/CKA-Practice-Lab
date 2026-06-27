# Q115. NetworkPolicy: Allow from IP Block 10.0.0.0/8, Deny 10.0.5.0/24

## Task
Create a NetworkPolicy that:
- Allows ingress from `10.0.0.0/8`
- Except from `10.0.5.0/24` (deny this subnet within the allowed block)

---

## Phase 1: Set Up the Environment

```bash
kubectl create namespace ipblock-test
kubectl run server --image=nginx -n ipblock-test --labels="app=server"
```

---

## Phase 2: Create the IP Block NetworkPolicy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ipblock
  namespace: ipblock-test
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8           # Allow this entire range
        except:
        - 10.0.5.0/24              # Except this sub-range (denied)
    ports:
    - protocol: TCP
      port: 80
EOF
```

**Field breakdown:**

| Field | Value | Meaning |
|---|---|---|
| `ipBlock.cidr` | `10.0.0.0/8` | Allow all IPs in 10.x.x.x range |
| `ipBlock.except` | `10.0.5.0/24` | Exclude IPs 10.0.5.0–10.0.5.255 from the allow |

**Effective result:**
- `10.0.1.x` → ✅ Allowed
- `10.0.4.x` → ✅ Allowed
- `10.0.5.x` → ❌ Denied (in except list)
- `10.0.6.x` → ✅ Allowed
- `192.168.x.x` → ❌ Denied (not in 10.0.0.0/8)

---

## Phase 3: Verify

```bash
kubectl get networkpolicy allow-ipblock -n ipblock-test
kubectl describe networkpolicy allow-ipblock -n ipblock-test
```

```
Spec:
  PodSelector: app=server
  Ingress:
  - From:
      IPBlock:
        CIDR: 10.0.0.0/8
        Except: [10.0.5.0/24]
    Ports: 80/TCP
  PolicyTypes: Ingress
```

---

## Phase 4: Test

```bash
SERVER_IP=$(kubectl get pod server -n ipblock-test -o jsonpath='{.status.podIP}')

# From an allowed IP (simulate with a pod — pod IPs are from pod CIDR, usually 10.x.x.x)
kubectl run allowed --image=busybox:1.28 --rm -it -- wget -qO- --timeout=3 $SERVER_IP:80
# Expected: 200 OK (pod is in 10.x.x.x range, not in except list)
```

> In a real lab with VMs, you'd test from actual hosts in those IP ranges.
> In a pod-based test, verify the policy config is correct via `kubectl describe`.

---

## Important: `ipBlock` Applies to External IPs

`ipBlock` is designed for **external traffic** (traffic originating outside the cluster).
- Pod-to-pod traffic uses `podSelector` and `namespaceSelector`
- Node IP traffic and external client traffic uses `ipBlock`

| Traffic Type | Use |
|---|---|
| Pod to pod | `podSelector` |
| Cross-namespace | `namespaceSelector` |
| External clients, nodes, VMs | `ipBlock` |

---

## Summary: Quick Reference

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.0.5.0/24
  ports:
  - port: 80
```

---

## Cleanup

```bash
kubectl delete namespace ipblock-test
```
