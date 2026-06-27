# Q118. NetworkPolicy: Allow Egress to Specific External IPs and Ports

## Task
Create a NetworkPolicy that allows pods to make egress connections to:
- External IP `203.0.113.50` on port `443`
- External IP `203.0.113.51` on port `5432`
- Block all other egress

---

## Phase 1: Set Up the Environment

```bash
kubectl create namespace external-egress
kubectl run app-pod --image=busybox:1.28 -n external-egress \
  --labels="app=myapp" -- sleep 3600
```

---

## Phase 2: Create the Egress NetworkPolicy

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
  namespace: external-egress
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  # Allow DNS (always required)
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
  # Allow HTTPS to specific external IP
  - to:
    - ipBlock:
        cidr: 203.0.113.50/32     # /32 = single IP
    ports:
    - protocol: TCP
      port: 443
  # Allow PostgreSQL to specific external IP
  - to:
    - ipBlock:
        cidr: 203.0.113.51/32
    ports:
    - protocol: TCP
      port: 5432
EOF
```

**Key points:**
- Use `/32` CIDR for a single IP address
- Use `/24`, `/16`, etc. for ranges
- Each `to:` block is a separate rule — they combine with OR logic
- Always include the DNS rule or name resolution will break

---

## Phase 3: Verify

```bash
kubectl get networkpolicy allow-external-egress -n external-egress
kubectl describe networkpolicy allow-external-egress -n external-egress
```

```
Spec:
  PodSelector: app=myapp
  Egress:
  - To:
      IPBlock: cidr 203.0.113.50/32
    Ports: 443/TCP
  - To:
      IPBlock: cidr 203.0.113.51/32
    Ports: 5432/TCP
  - To: (kube-dns)
    Ports: 53/UDP,53/TCP
  PolicyTypes: Egress
```

---

## Phase 4: Test

```bash
# Test allowed destination (should succeed)
kubectl exec -n external-egress app-pod -- \
  wget -qO- --timeout=3 https://203.0.113.50
# Expected: response (or connection refused — policy allowed it, server may not respond)

# Test blocked destination (should timeout)
kubectl exec -n external-egress app-pod -- \
  wget -qO- --timeout=3 http://8.8.8.8
# Expected: timeout — policy blocks this IP

# Verify DNS still works
kubectl exec -n external-egress app-pod -- nslookup kubernetes.default
# Expected: resolves successfully
```

---

## Combined ipBlock Example: Range With Exception

```yaml
egress:
- to:
  - ipBlock:
      cidr: 10.0.0.0/8        # Allow 10.x.x.x
      except:
      - 10.0.5.0/24           # But not 10.0.5.x
  ports:
  - port: 443
```

---

## Summary: Quick Reference

```yaml
# Single external IP
- to:
  - ipBlock:
      cidr: 203.0.113.50/32
  ports:
  - port: 443

# IP range
- to:
  - ipBlock:
      cidr: 192.168.0.0/16
  ports:
  - port: 5432
```

```bash
# Apply
kubectl apply -f allow-external-egress.yaml

# Verify
kubectl describe netpol allow-external-egress -n external-egress
```

---

## Cleanup

```bash
kubectl delete namespace external-egress
```
