# Q184–Q190. Advanced Cluster Operations and Debugging

## Q184: Optimize Cluster Performance
## Q185: Debug PV Provisioning in Cloud
## Q186: Complex Networking Debug (NetworkPolicy + Ingress + Service Mesh)
## Q187: Custom Scheduler Debug
## Q188: Complete Cluster Health Check
## Q189: Pods Evicted Due to Node Pressure
## Q190: Pod Security Standards Preventing Pod Creation

---

## Q184: Optimize Cluster Performance

```bash
# 1. Identify bottlenecks
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -10

# 2. Tune kubelet settings
cat /var/lib/kubelet/config.yaml
# Key tuning options:
# imageGCHighThresholdPercent: 85   (default: 85)
# imageGCLowThresholdPercent: 80    (default: 80)
# evictionHard:
#   memory.available: "200Mi"       (default: 100Mi — increase for stability)
#   nodefs.available: "10%"

# 3. Tune etcd
# Check etcd metrics
ETCDCTL_API=3 etcdctl endpoint status --write-out=table \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Defragment etcd (free disk space)
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 4. Reserve node resources for system
# In kubelet config:
# systemReserved:
#   cpu: 200m
#   memory: 500Mi
# kubeReserved:
#   cpu: 100m
#   memory: 200Mi
```

---

## Q185: Debug PV Provisioning in Cloud (AWS EBS Example)

```bash
# Step 1: Check StorageClass
kubectl get storageclass
kubectl describe storageclass gp2   # or fast-storage

# Step 2: Check PVC events
kubectl describe pvc my-pvc
# Events:
#   Warning  ProvisioningFailed  "InvalidVolume.NotFound: The volume ... does not exist"
#   Warning  ProvisioningFailed  "UnauthorizedOperation: You are not authorized to perform this operation"

# Step 3: Check cloud credentials (AWS)
# If using IRSA (IAM Roles for Service Accounts):
kubectl describe sa ebs-csi-controller-sa -n kube-system | grep Annotations
# Should have: eks.amazonaws.com/role-arn annotation

# Step 4: Check CSI driver pods
kubectl get pods -n kube-system | grep ebs-csi
kubectl logs -n kube-system -l app=ebs-csi-controller | tail -20

# Step 5: Validate IAM permissions
# Required: ec2:CreateVolume, ec2:AttachVolume, ec2:DescribeVolumes, ec2:DeleteVolume

# Step 6: Check volume plugin / CSI driver is installed
kubectl get csidrivers
# ebs.csi.aws.com   ← should exist
```

---

## Q186: Complex Networking Debug (Multi-Layer)

```bash
# Layer 1: Pod → Service (selector match?)
kubectl get endpoints my-service -n production

# Layer 2: NetworkPolicy (any deny?)
kubectl get networkpolicy -n production
kubectl describe networkpolicy -n production

# Layer 3: Service Mesh (if Istio)
# Check sidecar injection
kubectl get pods -n production -o yaml | grep istio-proxy
# Check virtual service
kubectl get virtualservice -n production
kubectl describe virtualservice my-app -n production

# Layer 4: Ingress rules
kubectl describe ingress my-ingress -n production

# Trace the full path with debug pod
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash
# Inside: curl, nmap, tcpdump, dig all available
curl -v http://my-service.production.svc.cluster.local:8080
dig my-service.production.svc.cluster.local
```

---

## Q187: Debug Custom Scheduler

```bash
# Deploy a custom scheduler
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: custom-scheduler
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: custom-scheduler
  template:
    metadata:
      labels:
        component: custom-scheduler
    spec:
      serviceAccountName: custom-scheduler
      containers:
      - name: custom-scheduler
        image: k8s.gcr.io/kube-scheduler:v1.28.0
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/custom-scheduler.yaml
EOF

# Configure a pod to use custom scheduler
kubectl run my-pod --image=nginx -o yaml --dry-run=client | \
  kubectl patch -f - --patch '{"spec":{"schedulerName":"custom-scheduler"}}' \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify scheduling decisions
kubectl get events -n default | grep custom-scheduler

# Debug: check custom scheduler logs
kubectl logs -n kube-system deploy/custom-scheduler
```

---

## Q188: Complete Cluster Health Check

```bash
#!/bin/bash
echo "=== CLUSTER HEALTH CHECK ==="

echo "--- Nodes ---"
kubectl get nodes

echo "--- Control Plane Components ---"
kubectl get pods -n kube-system | grep -E 'apiserver|scheduler|controller-manager|etcd'

echo "--- CoreDNS ---"
kubectl get pods -n kube-system -l k8s-app=kube-dns

echo "--- kube-proxy ---"
kubectl get pods -n kube-system -l k8s-app=kube-proxy

echo "--- CNI Plugin ---"
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'

echo "--- Resource Usage ---"
kubectl top nodes 2>/dev/null || echo "metrics-server not available"

echo "--- Unhealthy Pods ---"
kubectl get pods -A | grep -v -E 'Running|Completed'

echo "--- Certificates ---"
kubeadm certs check-expiration 2>/dev/null | grep -E 'CERTIFICATE|days|EXPIRED'

echo "--- etcd Health ---"
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

echo "=== CHECK COMPLETE ==="
```

---

## Q189: Pods Evicted Due to Node Pressure

```bash
# Check eviction events
kubectl get events -A | grep -i evict
# Warning  Evicted  pod/heavy-pod  The node was low on resource: memory

# Check node conditions causing eviction
kubectl describe node worker-1 | grep -A 5 Conditions
# MemoryPressure:  True   ← triggering eviction

# Check which pods were evicted
kubectl get pods -A | grep Evicted

# Clean up evicted pods
kubectl delete pods -A --field-selector=status.phase=Failed

# Prevent future evictions:
# 1. Add resource requests (so scheduler picks the right node)
kubectl set resources deployment/heavy-app --requests=memory=512Mi

# 2. Adjust kubelet eviction thresholds
# In /var/lib/kubelet/config.yaml:
# evictionHard:
#   memory.available: "300Mi"   ← increase threshold
# evictionSoft:
#   memory.available: "500Mi"
#   memory.available.gracePeriod: "2m"

# 3. Use PodDisruptionBudget to limit evictions
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: heavy-app
EOF
```

---

## Q190: Pod Security Standards Blocking Pod Creation

```bash
# Check namespace PSS labels
kubectl describe namespace restricted-ns | grep Labels
# pod-security.kubernetes.io/enforce: restricted

# Check the error when creating a pod
kubectl run test-pod --image=nginx -n restricted-ns
# Error: pods "test-pod" is forbidden:
#   violates PodSecurity "restricted:latest": ...
#   allowPrivilegeEscalation != false
#   unrestricted capabilities

# Fix: create a compliant pod spec
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:unprivileged
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
EOF

# To relax the policy on a namespace:
kubectl label namespace restricted-ns \
  pod-security.kubernetes.io/enforce=baseline --overwrite

# PSS levels:
# privileged → no restrictions
# baseline   → prevents privilege escalation
# restricted → most secure (requires runAsNonRoot, drop ALL caps, etc.)
```
