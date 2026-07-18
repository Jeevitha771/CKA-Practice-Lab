# Q90. Create ClusterIP Service `backend-service` with Session Affinity

## Task
Create a ClusterIP service named `backend-service` with:
- Selector: `app=backend` **and** `tier=api`
- Port: `8080`
- TargetPort: `8080`
- Session Affinity: `ClientIP`
- Session Affinity Timeout: `3600s`

---

## Phase 1: Deploy a Backend to Test Against

First, create a backing deployment so the service has real pods to route to.

```bash
kubectl create deployment backend --image=nginx --replicas=3
kubectl label deployment backend tier=api
kubectl label pods -l app=backend tier=api
```
NOTE: The Trap: Because Deployments are self-healing, if you delete one of these pods (or a node crashes), the ReplicaSet will immediately spin up a new pod using the original Pod Template. Because the template was never updated, the new pod will only have app=backend and will be missing tier=api.

> **Why:** A service with a selector but no matching pods will have empty endpoints ΓÇö
> it won't route any traffic. The deployment gives us real targets to verify against.

---

## Phase 2: Create the Service

### Option A ΓÇö Imperative (fast, exam-speed)

```bash
kubectl create service clusterip backend-service \
  --tcp=8080:80\
  --dry-run=client -o yaml > backend-service.yaml
```

The imperative command cannot set session affinity ΓÇö so we generate YAML and patch it.
Open `backend-service.yaml`, then add the selector and affinity fields (see Option B).

---

### Option B ΓÇö YAML (recommended ΓÇö full control)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
    tier: api
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
EOF
```

**Field breakdown:**

| Field | Value | Purpose |
|---|---|---|
| `type` | `ClusterIP` | Internal-only service ΓÇö no external IP |
| `selector.app` | `backend` | Match pods with label `app=backend` |
| `selector.tier` | `api` | **Both** labels must match (AND logic) |
| `port` | `8080` | Port the service listens on |
| `targetPort` | `8080` | Port on the pod the traffic is forwarded to |
| `sessionAffinity` | `ClientIP` | Route same client always to same pod |
| `timeoutSeconds` | `3600` | Keep the sticky session for 1 hour |

---

## Phase 3: Verify the Service

### Check service was created correctly

```bash
kubectl get service backend-service
```

Expected output:
```
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
backend-service   ClusterIP   10.96.x.x       <none>        8080/TCP   10s
```

### Inspect full details

```bash
kubectl describe service backend-service
```

Expected output (key fields):
```
Name:              backend-service
Namespace:         default
Selector:          app=backend,tier=api
Type:              ClusterIP
IP:                10.96.x.x
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Session Affinity:  ClientIP
SessionAffinityConfig:
  ClientIP:
    TimeoutSeconds: 3600
```

> **What to confirm:**
> - `Selector` shows **both** labels ΓÇö `app=backend,tier=api`
> - `Session Affinity: ClientIP` is present
> - `TimeoutSeconds: 3600` is correct

---

## Phase 4: Verify Endpoints Are Populated

A service only routes traffic if its endpoints are populated (i.e., matching pods exist and are Ready).

```bash
kubectl get endpoints backend-service
```

Expected output:
```
NAME              ENDPOINTS                                         AGE
backend-service   10.244.0.5:8080,10.244.1.3:8080,10.244.2.2:8080  30s
```

> **If ENDPOINTS shows `<none>`:** The selector labels don't match any running pods.
> Fix: verify pod labels with `kubectl get pods --show-labels` and check both
> `app=backend` AND `tier=api` are present.

---

## Phase 5: Verify Session Affinity Works

Session affinity means requests from the same client IP always go to the same pod.
We need to change the default Nginx index.html page in each pod to display its own unique hostname. Run this command on your control plane to loop through your backend pods and overwrite their HTML files:

```bash
for pod in $(kubectl get pods -l app=backend,tier=api -o jsonpath='{.items[*].metadata.name}'); do
  kubectl exec $pod -- sh -c 'echo "Server name: $(hostname)" > /usr/share/nginx/html/index.html'
done
```

```bash
# Run a temporary client pod
kubectl run client --image=busybox:1.28 --rm -it -- sh
```
NOTE: In above command, The --rm flag stands for remove.
When you combine --rm with an interactive shell session (-it -- sh), it tells Kubernetes to automatically delete the pod as soon as you exit the terminal.

Inside the shell, hit the service multiple times and observe the response:

```sh
# Replace 10.96.x.x with your actual ClusterIP
for i in $(seq 1 6); do
  wget -qO- 10.96.x.x:8080 2>/dev/null | grep "Server name"
done

or

while true; do wget -q -O- http://backend-service:8080; sleep 1; done
```

Expected output (same pod hostname repeating):
```
Server name: backend-79d8f9b4c-xrk2p
Server name: backend-79d8f9b4c-xrk2p
Server name: backend-79d8f9b4c-xrk2p
Server name: backend-79d8f9b4c-xrk2p
Server name: backend-79d8f9b4c-xrk2p
Server name: backend-79d8f9b4c-xrk2p
```

> All 6 requests go to the **same pod** ΓÇö that is session affinity working correctly.
> Without session affinity (default `None`), the requests would round-robin across all 3 pods.

---

## Phase 6: Verify via DNS (from inside a pod)

```bash
kubectl run dns-test --image=busybox:1.28 --rm -it -- sh
```

```sh
# Resolve the service by DNS name
nslookup backend-service

# Access using DNS name (same namespace)
wget -qO- backend-service:8080
```

Expected DNS output:
```
Server:    10.96.0.10
Name:      backend-service.default.svc.cluster.local
Address 1: 10.96.x.x
```

---

## How Session Affinity Works Internally

```
Client Pod (10.244.0.9)
         |
         Γû╝
   backend-service:8080  (ClusterIP: 10.96.x.x)
         |
   kube-proxy checks: "Have I seen 10.244.0.9 before?"
         |
    YES ΓåÆ always forward to 10.244.1.3:8080 (Pod-2)
    NO  ΓåÆ pick a pod, remember it for 3600s
         |
         Γû╝
   Pod-2 (10.244.1.3:8080)
```

kube-proxy implements session affinity using **iptables rules** that track source IP.
The `timeoutSeconds: 3600` value controls how long the mapping is kept before it expires.

---

## Common Mistakes & Fixes

| Mistake | Symptom | Fix |
|---|---|---|
| Selector uses only one label | Endpoints empty if pod doesn't have both labels | Add both `app` and `tier` to selector |
| Wrong `targetPort` | Traffic reaches service but pods return connection refused | Match `targetPort` to the actual container port |
| `sessionAffinity: None` (default) | Requests round-robin ΓÇö not sticky | Set `sessionAffinity: ClientIP` |
| Missing `sessionAffinityConfig` | Timeout defaults to 10800s (3 hours) | Add `timeoutSeconds: 3600` explicitly |
| Using `kubectl expose` | No way to set session affinity imperatively | Use YAML or patch after creation |

---

## Summary: Quick Reference

```bash
# Create (YAML is the only reliable way for session affinity)
kubectl apply -f backend-service.yaml

# Verify
kubectl get svc backend-service
kubectl describe svc backend-service
kubectl get endpoints backend-service

# Test from inside cluster
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- backend-service:8080

# Check labels on pods
kubectl get pods --show-labels -l app=backend
```

---

## Cleanup

```bash
kubectl delete service backend-service
kubectl delete deployment backend
kubectl delete pod client --ignore-not-found
kubectl delete pod dns-test --ignore-not-found
```
