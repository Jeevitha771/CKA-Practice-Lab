# Q91. Create NodePort Service `web-service`: Expose Deployment `web-app`

## Task
Create a NodePort service named `web-service` that:
- Exposes deployment `web-app`
- Service port: `80`
- Target port: `8080`
- NodePort: `30080`

Verify external access.

---

## Phase 1: Deploy `web-app`

```bash
kubectl create deployment web-app --image=nginx --replicas=2
```

The default nginx container listens on port **80**, but for this exercise we simulate an
app on **8080** by patching the container port (or just use the deployment as-is for
verification purposes):

```bash
# Verify pods are running
kubectl get pods -l app=web-app
```

---

## Phase 2: Create the NodePort Service

### Option A — Imperative (cannot set NodePort number directly)

```bash
kubectl expose deployment web-app \
  --name=web-service \
  --type=NodePort \
  --port=80 \
  --target-port=8080
```

> ⚠️ This assigns a **random** NodePort (30000–32767). To fix it to `30080`, use Option B.

---

### Option B — YAML (recommended for exam — controls NodePort value)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30080
EOF
```

**Field breakdown:**

| Field | Value | Purpose |
|---|---|---|
| `type` | `NodePort` | Opens a port on every node in the cluster |
| `selector.app` | `web-app` | Routes to pods with this label |
| `port` | `80` | ClusterIP port (internal service port) |
| `targetPort` | `8080` | Port on the pod container |
| `nodePort` | `30080` | Port opened on every node (external access) |

---

## Phase 3: Verify the Service

```bash
kubectl get service web-service
```

Expected output:
```
NAME          TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
web-service   NodePort   10.96.x.x     <none>        80:30080/TCP   10s
```

> The `80:30080/TCP` format means: internal port 80 is mapped to NodePort 30080.

```bash
kubectl describe service web-service
```

Key fields to confirm:
```
Type:                     NodePort
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.x.x:8080,10.244.x.x:8080
```

---

## Phase 4: Verify Endpoints Are Populated

```bash
kubectl get endpoints web-service
```

Expected:
```
NAME          ENDPOINTS                           AGE
web-service   10.244.0.5:8080,10.244.1.3:8080    30s
```

> If `<none>` — the selector `app=web-app` doesn't match pod labels. Run:
> ```bash
> kubectl get pods --show-labels -l app=web-app
> ```

---

## Phase 5: Verify External Access

### Get a Node IP

```bash
kubectl get nodes -o wide
```

```
NAME           STATUS   ROLES    INTERNAL-IP    EXTERNAL-IP
controlplane   Ready    master   192.168.1.10   <none>
worker-1       Ready    <none>   192.168.1.11   <none>
```

### Access via NodePort

```bash
# From inside the cluster or from the node itself
curl http://192.168.1.11:30080

# Or from the control plane node
curl http://192.168.1.10:30080
```

Expected:
```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
...
```

> ✅ `200 OK` response confirms external access via NodePort works.

### Access from inside the cluster (ClusterIP still works)

```bash
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- web-service:80
```

---

## How NodePort Works

```
External Client
      |
      ▼
Node IP : 30080  (any node — kube-proxy listens on all nodes)
      |
      ▼
ClusterIP : 10.96.x.x : 80  (iptables DNAT)
      |
      ▼
Pod IP : 8080  (actual container)
```

> Even if the pod is on `worker-1`, hitting `worker-2:30080` still works — kube-proxy
> on `worker-2` will forward the traffic to the correct pod.

---

## Summary: Key Commands

```bash
# Create
kubectl apply -f web-service.yaml

# Verify
kubectl get svc web-service
kubectl describe svc web-service
kubectl get endpoints web-service

# Get node IPs
kubectl get nodes -o wide

# Test externally
curl http://<NODE-IP>:30080

# Test internally
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- web-service:80
```

---

## Cleanup

```bash
kubectl delete service web-service
kubectl delete deployment web-app
```
