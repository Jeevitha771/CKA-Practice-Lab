# Q85. Cross-Namespace Pod Communication — Pod IPs, Pod DNS, Service DNS

## Task
Create two pods in namespaces `frontend` and `backend`. Verify communication using:
- Pod IPs
- Pod DNS names (`pod-ip.namespace.pod.cluster.local`)
- Service DNS

---

## Step 1: Create Namespaces and Pods

```bash
# Create namespaces
kubectl create namespace frontend
kubectl create namespace backend

# Create backend pod and expose it as a Service
kubectl run backend-pod --image=nginx -n backend
kubectl expose pod backend-pod --port=80 --name=backend-svc -n backend

# Create frontend pod (used as the test client)
kubectl run frontend-pod --image=nginx -n frontend
```

Wait for all pods to be `Running`:
```bash
kubectl get pods -n backend
kubectl get pods -n frontend
```

---

## Step 2: Get the Backend Pod IP

```bash
kubectl get pods -n backend -o wide
# Note the IP column — e.g. 10.244.1.5
```

---

## Step 3: Test 1 — Communicate Using Pod IP

```bash
# Replace YOUR_POD_IP with the IP from Step 2
kubectl exec -it frontend-pod -n frontend -- curl YOUR_POD_IP
```

**Expected:** `Welcome to nginx!` HTML output.  
**Why it works:** By default, all pods in a Kubernetes cluster can reach each other by IP regardless of namespace.

---

## Step 4: Test 2 — Communicate Using Pod DNS

Kubernetes assigns every pod an auto-generated DNS name based on its IP:

```
<dashed-ip>.<namespace>.pod.cluster.local
```

Replace dots `.` with dashes `-` in the IP (e.g. `10.244.1.5` → `10-244-1-5`):

```bash
kubectl exec -it frontend-pod -n frontend -- curl 10-244-1-5.backend.pod.cluster.local
```

**Expected:** `Welcome to nginx!` HTML output.  
**Caveat:** This name breaks if the pod restarts and gets a new IP. Avoid using it in production.

---

## Step 5: Test 3 — Communicate Using Service DNS (Recommended)

The Service DNS name is permanent even when pod IPs change.  
Formula: `<service-name>.<namespace>.svc.cluster.local`

```bash
kubectl exec -it frontend-pod -n frontend -- curl backend-svc.backend.svc.cluster.local
```

**Expected:** `Welcome to nginx!` HTML output.  
**Best practice:** Always use Service DNS in application configs — it survives pod crashes and rolling updates.

---

## DNS Name Summary

| Method | Format | Stable? |
|---|---|---|
| Pod IP | `10.244.1.5` | ❌ Changes on restart |
| Pod DNS | `10-244-1-5.backend.pod.cluster.local` | ❌ Tied to IP |
| Service DNS (short) | `backend-svc.backend` | ✅ |
| Service DNS (FQDN) | `backend-svc.backend.svc.cluster.local` | ✅ Preferred |

---

## Cleanup

```bash
kubectl delete pod frontend-pod -n frontend
kubectl delete pod backend-pod -n backend
kubectl delete svc backend-svc -n backend
kubectl delete namespace frontend backend
```
