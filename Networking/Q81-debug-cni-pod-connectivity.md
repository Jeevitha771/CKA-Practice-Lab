# Q81. Pod Cannot Reach Other Pods — Debug CNI

## Task
Pod cannot reach other pods. Debug CNI: check plugin pods status, verify configuration, test connectivity, check node routes.

## Step 1: Check CNI Plugin Pod Status

```bash
# List CNI plugin pods (look for calico, flannel, cilium, weave)
kubectl get pods -n kube-system -o wide | grep -iE 'calico|flannel|cilium|weave'

# All pods should be: Running, READY = 1/1
# Bad states: CrashLoopBackOff, Pending, Error

# Check logs of a failing CNI pod
kubectl logs -n kube-system <cni-pod-name>

# Describe for events
kubectl describe pod -n kube-system <cni-pod-name>
```

## Step 2: Verify CNI Configuration

```bash
# Check CNI config files on the node
ls -l /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist   # filename varies by plugin

# Verify each node has a unique Pod CIDR assigned
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# Check for NetworkPolicy that might be dropping traffic
kubectl get networkpolicies --all-namespaces
```

## Step 3: Test Connectivity

```bash
# Create destination pod
kubectl run dest-pod --image=nginx --labels="app=dest"
kubectl expose pod dest-pod --name=dest-service --port=80

# Create source pod
kubectl run source-pod --image=busybox:1.28 --command -- sleep 3600

# Get destination pod IP
kubectl get pod dest-pod -o wide

# Test IP connectivity (bypasses DNS — tests raw CNI routing)
kubectl exec -it source-pod -- ping <DEST_POD_IP>

# Test DNS (tests CoreDNS)
kubectl exec -it source-pod -- nslookup dest-service
```

### DNS Success vs Failure

```
# ✅ Success
Server:    10.96.0.10
Name:      dest-service.default.svc.cluster.local
Address:   10.105.12.34

# ❌ Failure
** server can't find dest-service: NXDOMAIN
```

## Step 4: Check Node Routes and Networking

```bash
# SSH into node, then:

# 1. Check IP forwarding is enabled (must be 1)
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1

# 2. Check routing table — should have routes to other nodes' Pod CIDRs
ip route
# Look for: 10.244.x.0/24 via <other-node-ip> (flannel)
# Or:       blackhole 10.244.x.0/24  (calico IPIP)

# 3. Check iptables for dropped packets
iptables-save | grep -i DROP
iptables-save | grep -i <destination-pod-ip>
```

## Connectivity Debugging Decision Tree

```
Pod can't reach another pod
        |
        ├── Ping by IP fails?
        │       └── CNI issue: check CNI pod logs, ip route, iptables
        │
        ├── Ping by IP works but service name fails?
        │       └── DNS issue: check CoreDNS pods, kubectl exec -- nslookup
        │
        └── Only cross-node fails?
                └── Routing issue: check ip route on nodes, IP forwarding
```

## Cleanup

```bash
kubectl delete pod source-pod dest-pod
kubectl delete svc dest-service
```
