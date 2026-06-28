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
# Troubleshooting Kubernetes EndpointSlices for External Services

This guide explains how to properly route traffic from a Kubernetes cluster to an external database using `Service` and `EndpointSlice` resources, including how to troubleshoot common connection issues and set up a "clean slate" demonstration environment.

## 1. The Port Name "Gotcha"

When linking an `EndpointSlice` to a `Service`, port names are optional **but strictly enforced if used**.

* **Single Port:** If your Service only exposes one port, you do not need to name it.
* **The Rule:** If you provide a `name` for a port in your `EndpointSlice` (e.g., `name: mysql`), it **must exactly match** a named port in your `Service`. 

If your Service has an unnamed port and your EndpointSlice has a named port, `kube-proxy` will fail to map the endpoints, and traffic will not route.

### Correct YAML Structure (Unnamed Ports)

```yaml
---
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
    # No name field here
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db-service-1
  labels:
    kubernetes.io/service-name: external-db
addressType: IPv4
ports:
  - protocol: TCP
    port: 5432
    # No name field here either. Note the '-' making it a list item!
endpoints:
  - addresses:
      - "192.168.1.100" # Your external IP



## Verify the SetUp:
> kubectl run db-test --image=busybox:1.28 --rm -it -- sh

> # Inside the busybox shell:
/ # nc -zv external-db 5432

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
