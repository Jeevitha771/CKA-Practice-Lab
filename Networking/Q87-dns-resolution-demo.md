# Q87. Demonstrate DNS Resolution — Same Namespace, Different Namespace, FQDN, Headless Service

## Task
Demonstrate DNS resolution for:
- Service in the same namespace
- Service in a different namespace
- Service FQDN
- Headless service pod (StatefulSet)

---

## Setup

```bash
# 1. Create two namespaces
kubectl create ns ns1
kubectl create ns ns2

# 2. Deploy a client pod in ns1 (busybox 1.28 has reliable nslookup)
kubectl run client -n ns1 --image=busybox:1.28 -- sleep 3600

# 3. Deploy a target service in the SAME namespace (ns1)
kubectl create deployment web-local -n ns1 --image=nginx
kubectl expose deployment web-local -n ns1 --name=svc-local --port=80

# 4. Deploy a target service in a DIFFERENT namespace (ns2)
kubectl create deployment web-remote -n ns2 --image=nginx
kubectl expose deployment web-remote -n ns2 --name=svc-remote --port=80

# 5. Deploy a Headless Service and StatefulSet (simulating a database) in ns2
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: svc-headless
  namespace: ns2
spec:
  clusterIP: None      # <-- This makes it headless
  selector:
    app: db
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: ns2
spec:
  serviceName: "svc-headless"
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

Wait for all pods to be `Running` before proceeding:
```bash
kubectl get pods -n ns1
kubectl get pods -n ns2
```

---

## Demo 1: Service in the Same Namespace

Because the client and `svc-local` are both in `ns1`, you only need the short name.  
CoreDNS auto-appends the local namespace via `/etc/resolv.conf` inside the pod.

```bash
kubectl exec -it client -n ns1 -- nslookup svc-local
```

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      svc-local
Address 1: 10.105.x.x svc-local.ns1.svc.cluster.local
```

---

## Demo 2: Service in a Different Namespace

The client is in `ns1`; the target is in `ns2`. Using just `svc-remote` will fail (NXDOMAIN).  
Append the namespace to the name:

```bash
kubectl exec -it client -n ns1 -- nslookup svc-remote.ns2
```

```
Name:      svc-remote.ns2
Address 1: 10.106.x.x svc-remote.ns2.svc.cluster.local
```

---

## Demo 3: Service FQDN (Fully Qualified Domain Name)

The absolute path to any service. Works from any namespace in the cluster.  
Formula: `<service>.<namespace>.svc.cluster.local`

```bash
kubectl exec -it client -n ns1 -- nslookup svc-remote.ns2.svc.cluster.local
```

> ✅ Use FQDNs in application configuration for maximum portability and safety.

---

## Demo 4: Headless Service — Returns Pod IPs Directly

A headless service (`clusterIP: None`) has **no virtual IP of its own**.  
Instead, DNS returns the **individual IPs of every matching pod**.

```bash
# Look up the headless service — returns multiple A records (one per pod)
kubectl exec -it client -n ns1 -- nslookup svc-headless.ns2
```

```
Name:      svc-headless.ns2.svc.cluster.local
Address 1: 192.168.x.x db-0.svc-headless.ns2.svc.cluster.local
Address 2: 192.168.x.x db-1.svc-headless.ns2.svc.cluster.local
```

```bash
# Look up a SPECIFIC StatefulSet pod by name — bypasses load balancing entirely
kubectl exec -it client -n ns1 -- nslookup db-0.svc-headless.ns2.svc.cluster.local
```

Formula for a specific StatefulSet pod:
```
<pod-name>.<headless-svc>.<namespace>.svc.cluster.local
```

> ✅ Used in databases (MySQL primary/replica, Cassandra nodes) where you must target a specific instance.

---

## DNS Name Formats Summary

| Scenario | DNS Name | Works from |
|---|---|---|
| Same namespace (short) | `svc-local` | Same namespace only |
| Different namespace | `svc-remote.ns2` | Anywhere |
| FQDN | `svc-remote.ns2.svc.cluster.local` | Anywhere (preferred) |
| Headless service (all pods) | `svc-headless.ns2.svc.cluster.local` | Anywhere |
| Specific StatefulSet pod | `db-0.svc-headless.ns2.svc.cluster.local` | Anywhere |

---

## How Search Domains Work

Every pod's `/etc/resolv.conf` is automatically populated by kubelet:
```
search ns1.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
```
CoreDNS appends each search domain suffix until the name resolves.  
That is why `svc-local` resolves — it expands to `svc-local.ns1.svc.cluster.local` automatically.

---

## Cleanup

```bash
kubectl delete ns ns1 ns2
```
