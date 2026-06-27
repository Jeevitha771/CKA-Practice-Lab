# Q97. Configure Service with `externalTrafficPolicy: Local`

## Task
Configure a NodePort (or LoadBalancer) service with `externalTrafficPolicy: Local`.
Explain the difference from the default `Cluster` policy.

---

## Phase 1: Deploy the Application

```bash
kubectl create deployment web --image=nginx --replicas=3
```

---

## Phase 2: Create Service with `externalTrafficPolicy: Local`

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-local
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30081
  externalTrafficPolicy: Local
EOF
```

---

## Phase 3: Understand the Difference

### Default: `externalTrafficPolicy: Cluster`

```
External Client (1.2.3.4)
         |
         ▼
   Node-A : 30081  (pod NOT running here)
         |
   kube-proxy SNATs the packet: src IP changed to Node-A IP
         |
         ▼
   Pod on Node-B  (sees Node-A IP, not 1.2.3.4)
```

**Cluster policy behaviour:**
- Traffic can be forwarded to any node that has a matching pod — even a different node.
- Source IP is **SNAT'd** (replaced with the node IP) so the pod cannot see the real client IP.
- All nodes accept traffic regardless of whether they have a local pod.

---

### `externalTrafficPolicy: Local`

```
External Client (1.2.3.4)
         |
         ▼
   Node-A : 30081  (pod IS running here)
         |
   No SNAT — original src IP preserved
         |
         ▼
   Pod on Node-A  (sees real client IP: 1.2.3.4)
```

**Local policy behaviour:**
- Traffic is only sent to pods running on the **receiving node**.
- If no pod runs on that node → connection is **dropped** (healthcheck returns unhealthy).
- Source IP is **preserved** — the pod sees the real client IP.
- Cloud LBs use health checks to only route to nodes that have local pods.

---

## Phase 4: Verify

```bash
kubectl get service web-local
```

```
NAME        TYPE       CLUSTER-IP    PORT(S)        AGE
web-local   NodePort   10.96.x.x     80:30081/TCP   10s
```

```bash
kubectl describe service web-local | grep -i traffic
```

```
External Traffic Policy:  Local
```

### Check which nodes have pods

```bash
kubectl get pods -l app=web -o wide
```

```
NAME              READY   NODE       IP
web-abc-111       1/1     worker-1   10.244.1.5
web-abc-222       1/1     worker-2   10.244.2.3
web-abc-333       1/1     worker-1   10.244.1.6
```

> With `Local` policy:
> - `worker-1:30081` → routes to its 2 local pods ✅
> - `worker-2:30081` → routes to its 1 local pod ✅
> - `controlplane:30081` → **drops traffic** (no pod here) ❌

---

## Cluster vs Local — Comparison Table

| Aspect | `Cluster` (default) | `Local` |
|---|---|---|
| Traffic routing | Any node → any pod (cross-node OK) | Only to pods on the receiving node |
| Source IP preserved | ❌ No — SNAT replaces client IP | ✅ Yes — real client IP visible to pod |
| Nodes with no local pod | Accept traffic, forward elsewhere | Drop traffic (LB health check fails) |
| Load balance uniformity | Even across all pods | Uneven if pods distributed unevenly |
| Use case | General workloads | IP-based access control, audit logging, rate limiting |

---

## When to Use `Local`

Use `externalTrafficPolicy: Local` when:
- Your application needs the **real client IP** (access logs, geo-blocking, rate limiting)
- You are running on a cloud provider where the LB respects health checks
- Your pods are evenly distributed across nodes (avoid uneven load)

Do NOT use `Local` when:
- You have more nodes than pods (some nodes drop all traffic)
- Traffic distribution uniformity matters more than source IP

---

## Summary: Quick Reference

```bash
# Create with Local policy
kubectl apply -f web-local.yaml

# Verify policy
kubectl describe svc web-local | grep -i "traffic policy"

# See which nodes have pods
kubectl get pods -l app=web -o wide

# Test (hit a node that HAS a pod)
NODE_IP=$(kubectl get node worker-1 -o jsonpath='{.status.addresses[0].address}')
curl http://$NODE_IP:30081
```

---

## Cleanup

```bash
kubectl delete service web-local
kubectl delete deployment web
```
