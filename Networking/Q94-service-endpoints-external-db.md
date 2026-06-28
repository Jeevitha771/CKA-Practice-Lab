# Q94. Manually Create Service and Endpoints for External Database

## Task
Create a service and manual Endpoints object for an **external** database (running outside Kubernetes):
- Service name: `external-db`
- External IP: `192.168.1.100`
- Port: `5432`
- No selector (manual endpoint management)

---

## Why This Pattern Exists

By default, a Kubernetes Service uses a **selector** to automatically discover and track pod IPs
via Endpoints. When the backend is **outside** the cluster (legacy database, on-prem server, 
external SaaS), there are no pods to select. You manually create the Endpoints object yourself.

```
Pod inside cluster
      |
      ▼
Service: external-db (10.96.x.x:5432)  ← no selector
      |
      ▼
Endpoints: external-db (192.168.1.100:5432)  ← manually defined
      |
      ▼
External database server (192.168.1.100:5432)
```

---

## Phase 1: Create the Service (No Selector)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  # Note: NO selector field — endpoints are managed manually
EOF
```

> Without a `selector`, Kubernetes will **not** auto-populate the Endpoints object.
> The service gets a ClusterIP but has no endpoints until we add them manually.

---

## Phase 2: Create the Endpoints Object
**NEW Approach:**
Create an EndpointSlice object.
Crucial Rule: Unlike the old API that relied on identical names, EndpointSlice links to your Service using a specific label: kubernetes.io/service-name.
**Create a file named external-db-endpointslice.yaml:**

```bash
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db-service-1 # Can be named anything, usually suffixed with a number
  labels:
    # THIS IS THE CRUCIAL LINK TO YOUR SERVICE
    kubernetes.io/service-name: external-db
addressType: IPv4
ports:
  - name: mysql          # Must match the port name in the Service
    protocol: TCP
    port: 3306
endpoints:
  - addresses:
      - "198.51.100.24"  # The actual IP address of your external database
```
**Apply it:**

> kubectl apply -f external-db-endpointslice.yaml

**OLD Approach:**
The Endpoints object must have the **same name** as the Service.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db     # Must exactly match the Service name
subsets:
- addresses:
  - ip: 192.168.1.100   # External database server IP
  ports:
  - port: 5432
    protocol: TCP
EOF
```

> ⚠️ The `metadata.name` of Endpoints **must** match the Service name exactly.
> Kubernetes links them by name — not by any label or reference field.

---

## Phase 3: Verify

### Check service

```bash
kubectl get service external-db
```

```
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
external-db   ClusterIP   10.96.x.x     <none>        5432/TCP   20s
```

### Check endpoints

```bash
kubectl get endpoints external-db
```

```
NAME          ENDPOINTS             AGE
external-db   192.168.1.100:5432    20s
```

> Unlike selector-based services, these endpoints won't change — Kubernetes doesn't
> manage them. You are responsible for keeping the Endpoints object updated.

### Describe for full details

```bash
kubectl describe endpoints external-db
```

```
Name:         external-db
Namespace:    default
Subsets:
  Addresses: 192.168.1.100
  Ports:
    Port: 5432/TCP
```

---

## Phase 4: Test Connectivity from Inside the Cluster

```bash
kubectl run db-test --image=busybox:1.28 --rm -it -- sh
```

Inside the shell:

```sh
# Test DNS resolution (service name → ClusterIP)
nslookup external-db

# Test TCP connectivity to the external server via the service
# (requires nc / telnet — busybox has nc)
nc -zv external-db 5432
```

Expected:
```
external-db.default.svc.cluster.local resolved to 10.96.x.x
Connection to external-db 5432 port [tcp/postgresql] succeeded!
```

> Pods now connect to `external-db:5432` exactly as if it were an internal service.
> The actual traffic is forwarded to `192.168.1.100:5432` transparently.

---

## Alternative: ExternalName Service
For DNS-based external services (when you have a hostname instead of an IP):
> kubectl create service externalname my-ns --external-name bar.com


or
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-db-dns
spec:
  type: ExternalName
  externalName: db.corp.example.com   # CNAME target
EOF
```

| Approach | When to Use |
|---|---|
| Manual Endpoints | External IP is fixed and known |
| ExternalName | External hostname (DNS CNAME) |
| Regular selector service | Backend is a pod inside the cluster |

---

## Summary: Quick Reference

```bash
# Create service (no selector)
kubectl apply -f external-db-service.yaml

# Create endpoints manually
kubectl apply -f external-db-endpoints.yaml

# Verify
kubectl get svc external-db
kubectl get endpoints external-db

# Test from inside cluster
kubectl run test --image=busybox:1.28 --rm -it -- nc -zv external-db 5432
```
## Why is this so useful? (The Benefit)

Clean Configuration: Your application deployment inside Kubernetes simply connects to external-db-service on port 3306. It doesn't know (or care) that the database is external.

Easy Migrations: If you ever migrate that database to a different external server (meaning the IP changes), you do not need to restart your application or change its environment variables. You simply update the IP address in the EndpointSlice YAML and apply it. Kubernetes instantly redirects the traffic.

Bringing it In-House: If you eventually decide to move the database inside the Kubernetes cluster, you just delete the manual EndpointSlice, add a selector to the Service, and it seamlessly connects to your new database Pods without your app ever knowing.

---

## Cleanup

```bash
kubectl delete service external-db
kubectl delete endpoints external-db
```
