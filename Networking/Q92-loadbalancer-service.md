# Q92. Create LoadBalancer Service for `public-api`: Port 443 → 8443

## Task
Create a LoadBalancer service named `public-api-service` for deployment `public-api`:
- Port: `443`
- Target port: `8443`
- Protocol: `TCP`

Explain the difference vs NodePort. Verify external IP (cloud).

---

## Phase 1: Deploy `public-api`

```bash
kubectl create deployment public-api --image=nginx --replicas=2
```

```bash
kubectl get pods -l app=public-api
```

---

## Phase 2: Create the LoadBalancer Service

### Option A — Imperative

```bash
kubectl expose deployment public-api \
  --name=public-api-service \
  --type=LoadBalancer \
  --port=443 \
  --target-port=8443
```

### Option B — YAML (recommended)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: public-api-service
spec:
  type: LoadBalancer
  selector:
    app: public-api
  ports:
  - protocol: TCP
    port: 443
    targetPort: 8443
EOF
```

**Field breakdown:**

| Field | Value | Purpose |
|---|---|---|
| `type` | `LoadBalancer` | Requests an external LB from the cloud provider |
| `port` | `443` | HTTPS port exposed externally |
| `targetPort` | `8443` | Port the container listens on |

---

## Phase 3: Verify the Service

```bash
kubectl get service public-api-service
```

### In a cloud environment (GKE, EKS, AKS):

```
NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)         AGE
public-api-service   LoadBalancer   10.96.x.x     34.123.45.67     443:3xxxx/TCP   60s
```

> The `EXTERNAL-IP` column shows the cloud load balancer's public IP.
> Traffic to `34.123.45.67:443` is forwarded to pod port `8443`.

### In a bare-metal / kubeadm cluster (no cloud):

```
NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
public-api-service   LoadBalancer   10.96.x.x     <pending>     443:3xxxx/TCP   60s
```

> `<pending>` is expected — bare-metal clusters have no cloud controller to assign an IP.
> Use **MetalLB** to get LoadBalancer IPs on bare-metal, or use NodePort for the exam lab.

---

## Phase 4: Verify External Access (Cloud)

```bash
# Get the external IP
EXTERNAL_IP=$(kubectl get svc public-api-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $EXTERNAL_IP

# Test HTTPS (nginx won't have a real cert, so use -k to skip TLS verification)
curl -k https://$EXTERNAL_IP:443
```

---

## LoadBalancer vs NodePort — Key Differences

| Aspect | NodePort | LoadBalancer |
|---|---|---|
| External access | `<NodeIP>:<NodePort>` | Dedicated external IP/hostname |
| Port range | 30000–32767 only | Any port (443, 80, etc.) |
| Load balancing | Client must pick a node | Cloud LB distributes across nodes |
| Single point of failure | Yes (if client targets one node) | No (LB health-checks nodes) |
| Cloud requirement | None | Requires cloud provider or MetalLB |
| Cost | Free | Cloud charges per LB ($$) |
| Exam environment | ✅ Works in bare-metal lab | ⚠️ EXTERNAL-IP stays `<pending>` |

> **For the CKA exam:** LoadBalancer questions usually ask you to *create* the service and
> explain the concept — not to verify an external IP (since exam clusters are bare-metal).

---

## How LoadBalancer Works

```
External Client
      |
      ▼
Cloud Load Balancer (34.123.45.67:443)
      |  (health-checks multiple nodes)
      ▼
Node IP : <random NodePort>  (cloud LB picks a healthy node)
      |
      ▼
kube-proxy → ClusterIP → Pod:8443
```

> A LoadBalancer service automatically creates a NodePort **and** a ClusterIP under the hood.
> The cloud LB is just a managed front-end that sits in front of the NodePorts.

---

## Summary: Quick Reference

```bash
# Create
kubectl apply -f public-api-service.yaml

# Verify
kubectl get svc public-api-service
kubectl describe svc public-api-service

# Get external IP (cloud)
kubectl get svc public-api-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Test (cloud)
curl -k https://<EXTERNAL-IP>:443
```

---

## Cleanup

```bash
kubectl delete service public-api-service
kubectl delete deployment public-api
```
