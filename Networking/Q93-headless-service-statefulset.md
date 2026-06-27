# Q93. Create Headless Service for StatefulSet `database`

## Task
Create a headless service (ClusterIP: None) with:
- Service name: `database-headless`
- Selector: `app=database`
- Port: `5432`

Verify DNS records are created per pod.

---

## Phase 1: Deploy the StatefulSet

A headless service must be created **before** the StatefulSet that references it.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database-headless    # must match the headless service name
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "testpass"
EOF
```

---

## Phase 2: Create the Headless Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None          # This is what makes it headless
  selector:
    app: database
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
EOF
```

**Key field:**

| Field | Value | Purpose |
|---|---|---|
| `clusterIP` | `None` | No virtual IP — DNS resolves directly to pod IPs |
| `selector` | `app: database` | Which pods are part of this service |
| `port` | `5432` | PostgreSQL port |

---

## Phase 3: Verify the Service

```bash
kubectl get service database-headless
```

Expected output:
```
NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
database-headless   ClusterIP   None         <none>        5432/TCP   10s
```

> `CLUSTER-IP: None` confirms it is headless — no virtual IP assigned.

```bash
kubectl describe service database-headless
```

Key fields:
```
Name:              database-headless
Selector:          app=database
Type:              ClusterIP
IP:                None
Port:              <unset>  5432/TCP
Endpoints:         10.244.x.x:5432,10.244.x.x:5432,10.244.x.x:5432
```

---

## Phase 4: Verify Per-Pod DNS Records

This is the key feature of a headless service — each pod gets its own DNS A record.

### Pod DNS format (StatefulSet):
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For our 3-pod StatefulSet:
```
database-0.database-headless.default.svc.cluster.local → 10.244.x.x
database-1.database-headless.default.svc.cluster.local → 10.244.x.x
database-2.database-headless.default.svc.cluster.local → 10.244.x.x
```

### Test DNS from inside the cluster:

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it -- sh
```

Inside the shell:

```sh
# Resolve the service — returns ALL pod IPs
nslookup database-headless

# Resolve a specific pod directly
nslookup database-0.database-headless.default.svc.cluster.local
nslookup database-1.database-headless.default.svc.cluster.local
nslookup database-2.database-headless.default.svc.cluster.local
```

Expected output for `nslookup database-headless`:
```
Server:    10.96.0.10
Name:      database-headless.default.svc.cluster.local
Address 1: 10.244.0.8   database-0.database-headless.default.svc.cluster.local
Address 2: 10.244.1.5   database-1.database-headless.default.svc.cluster.local
Address 3: 10.244.2.3   database-2.database-headless.default.svc.cluster.local
```

> DNS returns all 3 pod IPs directly — no load balancing, no virtual IP.
> The client application decides which pod to connect to.

Expected output for `nslookup database-0.database-headless.default.svc.cluster.local`:
```
Name:      database-0.database-headless.default.svc.cluster.local
Address 1: 10.244.0.8
```

> Each pod has a **stable, predictable DNS name** — even after pod restarts.
> This is critical for databases where the primary (database-0) must be addressed directly.

---

## Headless vs Regular ClusterIP Service

| Aspect | Regular ClusterIP | Headless (ClusterIP: None) |
|---|---|---|
| Virtual IP | Yes (`10.96.x.x`) | No — `None` |
| DNS resolves to | ClusterIP → kube-proxy → pod | Pod IPs directly |
| Load balancing | kube-proxy round-robins | Client-side (application controls) |
| Per-pod DNS | ❌ No | ✅ Yes — `pod-0.svc.ns.svc.cluster.local` |
| Use case | Stateless apps | StatefulSets, databases, Kafka, Zookeeper |

---

## Why StatefulSets Require Headless Services

```
database-0 → Primary (writes)
database-1 → Replica (reads)
database-2 → Replica (reads)
```

An application needs to connect to `database-0` specifically for writes and
`database-1` or `database-2` for reads. A regular ClusterIP would randomly
route to any pod — breaking replication logic.

With a headless service, the app can resolve each pod by name and connect directly.

---

## Summary: Quick Reference

```bash
# Create headless service
kubectl apply -f database-headless.yaml

# Verify
kubectl get svc database-headless        # CLUSTER-IP should be None
kubectl get endpoints database-headless  # All pod IPs listed

# Test per-pod DNS
kubectl run dns-test --image=busybox:1.28 --rm -it -- \
  nslookup database-0.database-headless.default.svc.cluster.local
```

---

## Cleanup

```bash
kubectl delete service database-headless
kubectl delete statefulset database
```
