# Q89. Test and Verify Pod-to-Pod Communication Across Different Nodes

## Task
Force two pods onto different nodes using `podAntiAffinity`, then verify cross-node
communication using `ping` (ICMP) and `curl` (TCP/HTTP).

---

## Phase 1: Deploy Pods on Different Nodes

We use `podAntiAffinity` to guarantee the scheduler places each replica on a **separate node**.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cross-node-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cross-node
  template:
    metadata:
      labels:
        app: cross-node
    spec:
      affinity:
        podAntiAffinity:
          # "Do not schedule me on a node that already has a pod with app=cross-node"
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cross-node
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx
EOF
```

Wait for both pods to be `Running`:

```bash
kubectl rollout status deployment cross-node-test
```

---

## Phase 2: Verify Node Placement and Get Pod IPs

```bash
kubectl get pods -l app=cross-node -o wide
```

Sample output:
```
NAME                              READY   STATUS    IP            NODE
cross-node-test-abc-xyz           1/1     Running   10.244.0.5    controlplane
cross-node-test-def-uvw           1/1     Running   10.244.1.4    node01
```

**What to check:**
- `NODE` column — two **different** node names confirm physical separation.
- `IP` column — IPs are from **different subnets** (e.g. `10.244.0.x` vs `10.244.1.x`),
  because each node owns a slice of the Pod CIDR.

**Action:** Pick one pod as **Sender** (copy its `NAME`) and the other as **Receiver** (copy its `IP`).

---

## Phase 3: Ping Test — ICMP Reachability

```bash
# Replace <SENDER_POD_NAME> and <RECEIVER_POD_IP> with your actual values
kubectl exec -it <SENDER_POD_NAME> -- ping -c 3 <RECEIVER_POD_IP>
```

Expected output:
```
PING 10.244.1.4 (10.244.1.4): 56 data bytes
64 bytes from 10.244.1.4: icmp_seq=0 ttl=62 time=0.52 ms
64 bytes from 10.244.1.4: icmp_seq=1 ttl=62 time=0.48 ms
64 bytes from 10.244.1.4: icmp_seq=2 ttl=62 time=0.50 ms

--- 10.244.1.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

**Why it works:** The CNI plugin (Calico, Flannel, Cilium, etc.) encapsulates the packet,
sends it across the physical network to the destination node, unwraps it, and delivers it
directly to the pod — all transparently.

---

## Phase 4: Curl Test — TCP/HTTP Traffic

`ping` tests raw ICMP reachability. `curl` proves that full TCP connections work across nodes.

```bash
# -I fetches HTTP headers only (keeps output clean)
kubectl exec -it <SENDER_POD_NAME> -- curl -I <RECEIVER_POD_IP>
```

Expected output:
```
HTTP/1.1 200 OK
Server: nginx/1.25.x
Date: ...
Content-Type: text/html
Content-Length: 615
```

**Why it matters:** A `200 OK` response proves that multi-packet TCP sessions traverse the
physical cluster boundary without any manual router configuration. The CNI handles all
overlay/underlay routing automatically.

---

## How Cross-Node Routing Works

```
[Sender Pod]  →  veth  →  [Node A bridge/tunl0]
                                    |
                          Physical network (eth0)
                                    |
                          [Node B bridge/tunl0]  →  veth  →  [Receiver Pod]
```

| CNI Plugin | Mechanism | Virtual Interface |
|---|---|---|
| Flannel | VXLAN overlay | `flannel.1` |
| Calico (IPIP) | IP-in-IP encapsulation | `tunl0` |
| Calico (BGP) | Native routing via BGP | `eth0` (no overlay) |
| Cilium | eBPF + VXLAN | `cilium_vxlan` |

Verify the route on a node:
```bash
# SSH into Sender node, then:
ip route
# Look for a line like: 10.244.1.0/24 via <node-B-IP> dev tunl0  (Calico IPIP)
# or:                   10.244.1.0/24 via <node-B-IP> dev flannel.1 (Flannel)
```

---

## Cleanup

```bash
kubectl delete deployment cross-node-test
```
