# Q167–Q174. Network Troubleshooting

## Q167: Pods Cannot Communicate Across Nodes
## Q168: Service Not Load Balancing Traffic
## Q169: Ingress Returning 404 Errors
## Q170: DNS Resolution Failing for Services
## Q171: External Traffic Cannot Reach NodePort
## Q172: NetworkPolicy Blocking Required Traffic
## Q173: Pods Reach Services by IP but Not DNS
## Q174: Intermittent Network Connectivity

---

## Q167: Pods Cannot Communicate Across Nodes

```bash
# Step 1: Verify CNI plugin is running
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'
# All should be Running and Ready

# Step 2: Test cross-node connectivity
POD_A=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
kubectl exec pod-b -- ping -c 3 $POD_A

# Step 3: Check node routes
ip route show
# should see 10.244.x.x routes via node IPs

# Step 4: Check CNI plugin logs
kubectl logs -n kube-system -l k8s-app=calico-node | grep -i error

# Step 5: Verify IP forwarding
sysctl net.ipv4.ip_forward
# = 1

# Step 6: Check for firewall dropping cross-node traffic
iptables -L FORWARD | grep DROP
# Fix: allow FORWARD
iptables -P FORWARD ACCEPT
```

---

## Q168: Service Not Load Balancing

```bash
# Step 1: Check service selector vs pod labels
kubectl get service web-service -o yaml | grep -A 5 selector
kubectl get pods --show-labels | grep app=web-app

# Step 2: Check endpoints
kubectl get endpoints web-service
# If empty — selector mismatch or pods not Ready

# Step 3: Check pod readiness
kubectl get pods -l app=web-app
# READY 0/1 = readiness probe failing → endpoints excluded

# Step 4: Check kube-proxy iptables rules
kubectl get service web-service
# CLUSTER-IP: 10.96.50.100
iptables -t nat -L KUBE-SERVICES | grep 10.96.50.100

# Fix selector mismatch
kubectl patch service web-service -p '{"spec":{"selector":{"app":"web-app"}}}'
```

---

## Q169: Ingress Returning 404

```bash
# Step 1: Check ingress controller pods
kubectl get pods -n ingress-nginx
# Running?

# Step 2: Check ingress controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep 404

# Step 3: Check ingress resource config
kubectl describe ingress my-ingress -n production
# Verify: host matches, paths correct, backend service names/ports correct

# Step 4: Check backend service
kubectl get service web-service -n production
kubectl get endpoints web-service -n production  # must not be empty

# Step 5: Check ingress class
kubectl get ingress my-ingress -n production -o yaml | grep ingressClassName
# Must match: kubectl get ingressclass
```

---

## Q170: DNS Resolution Failing

```bash
# Step 1: Test from a pod
kubectl run dns-test --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
# If fails: CoreDNS problem

# Step 2: CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Step 3: Pod resolv.conf
kubectl exec <app-pod> -- cat /etc/resolv.conf
# nameserver 10.96.0.10 ← must be CoreDNS ClusterIP

# Step 4: CoreDNS service
kubectl get service kube-dns -n kube-system
# CLUSTER-IP: 10.96.0.10 ← must match resolv.conf nameserver

# Step 5: NetworkPolicy blocking port 53
kubectl get networkpolicy -A
# Fix: ensure pods can reach kube-dns on UDP/TCP 53
```

---

## Q171: External Traffic Cannot Reach NodePort

```bash
# Step 1: Confirm service is NodePort and get port
kubectl get service web-service
# TYPE: NodePort   PORT(S): 80:30080/TCP

# Step 2: Test from external host
curl http://<node-ip>:30080

# Step 3: Check firewall on the node
iptables -L INPUT | grep 30080
# If not found: add rule
iptables -A INPUT -p tcp --dport 30080 -j ACCEPT

# Step 4: In cloud (AWS/GCP/Azure)
# Check security groups / firewall rules allow NodePort range 30000-32767

# Step 5: kube-proxy iptables
iptables -t nat -L KUBE-NODEPORTS | grep 30080
```

---

## Q172: NetworkPolicy Blocking Required Traffic

```bash
# Step 1: Identify the policy
kubectl get networkpolicy -n <namespace>

# Step 2: Check which policy is blocking
kubectl describe networkpolicy <policy-name> -n <namespace>

# Step 3: Test connectivity
kubectl exec pod-a -- wget -qO- --timeout=3 http://pod-b-ip:8080

# Step 4: Add exception to allow required traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-exception
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF
```

---

## Q173: Pods Reach by IP but Not DNS

```bash
# IP works:
kubectl exec app-pod -- wget -qO- http://10.96.50.100:80   # OK

# DNS fails:
kubectl exec app-pod -- wget -qO- http://web-service:80    # fails

# Debug resolv.conf
kubectl exec app-pod -- cat /etc/resolv.conf
# Is nameserver pointing to CoreDNS?
# Does search include svc.cluster.local?

# Test DNS directly
kubectl exec app-pod -- nslookup web-service 10.96.0.10
# If works with explicit DNS IP: pod's resolv.conf is wrong

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## Q174: Intermittent Network Connectivity

```bash
# Step 1: Repeated connectivity tests
for i in {1..20}; do
  kubectl exec test-pod -- wget -qO- --timeout=2 http://web-service:80 \
    && echo "OK" || echo "FAIL"
done

# Step 2: Check pod restarts (crashing pods = intermittent)
kubectl get pods -n <namespace> --sort-by='.status.containerStatuses[0].restartCount'

# Step 3: Endpoints changing (scale events)
kubectl get endpoints web-service -w
# Watch if endpoints flip on/off

# Step 4: Check node-level network issues
ping -c 10 <other-node-ip>
# Packet loss? Physical network issue

# Step 5: Review CNI logs for errors/warnings
kubectl logs -n kube-system -l k8s-app=calico-node | grep -i "error\|warning"
```
