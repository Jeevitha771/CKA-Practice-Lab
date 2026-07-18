# Q13. Install Calico CNI and Verify Cluster Networking

## Task
Install Calico CNI plugin and verify:
- All Calico pods are running
- Nodes show Ready status
- Pod-to-pod communication works across nodes

---

## Phase 1: Install Calico

```bash
# Apply the Calico manifest (compatible with pod CIDR 10.244.0.0/16 or 192.168.0.0/16)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

> If your pod CIDR is NOT `192.168.0.0/16` (Calico's default), patch it:
>
> ```bash
> # Download manifest first, then edit CALICO_IPV4POOL_CIDR
> curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
>
> # Edit the CIDR to match your kubeadm --pod-network-cidr
> sed -i 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|' calico.yaml
> sed -i 's|#   value: "192.168.0.0/16"|  value: "10.244.0.0/16"|' calico.yaml
>
> kubectl apply -f calico.yaml
> ```

---

## Phase 2: Verify Calico Pods Running

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```

```
NAME                READY   STATUS    RESTARTS   AGE
calico-node-abc12   1/1     Running   0          2m
calico-node-def34   1/1     Running   0          2m
```

```bash
# All calico-system pods (if using operator install)
kubectl get pods -n calico-system
```

---

## Phase 3: Verify Nodes are Ready

```bash
kubectl get nodes
```

```
NAME        STATUS   ROLES           AGE
master-1    Ready    control-plane   10m
worker-1    Ready    <none>          5m
worker-2    Ready    <none>          5m
```

---

## Phase 4: Verify Pod-to-Pod Communication

```bash
# Deploy two test pods on different nodes
kubectl run pod-a --image=busybox:1.28 -- sleep 3600
kubectl run pod-b --image=busybox:1.28 -- sleep 3600

# Wait for pods to be running
kubectl get pods -o wide
# NAME    READY   STATUS    NODE
# pod-a   1/1     Running   worker-1
# pod-b   1/1     Running   worker-2

# Get pod-b IP
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')

# Ping from pod-a to pod-b (cross-node)
kubectl exec pod-a -- ping -c 3 $POD_B_IP
```

Expected:
```
3 packets transmitted, 3 received, 0% packet loss
```

---

## Troubleshoot Calico

```bash
# Check Calico node logs
kubectl logs -n kube-system -l k8s-app=calico-node

# Verify BGP peers (Calico CLI)
kubectl exec -n kube-system calico-node-<id> -- calicoctl node status

# Check IP pool
kubectl get ippools -o yaml   # if using Calico operator
```

---

## Cleanup

```bash
kubectl delete pod pod-a pod-b
```
