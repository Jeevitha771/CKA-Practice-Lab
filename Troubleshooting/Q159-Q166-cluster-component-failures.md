# Q159–Q166. Cluster Component Failure Troubleshooting

## Q159: API Server Not Responding
## Q160: Scheduler Not Scheduling Pods
## Q161: Controller Manager Not Creating Replica Pods
## Q162: Kubelet Not Registering with Cluster
## Q163: CoreDNS Pods Not Running
## Q164: kube-proxy Not Updating iptables
## Q165: etcd Cluster Unhealthy
## Q166: Certificate Expiration

---

## Q159: API Server Not Responding

```bash
# Check if the API server static pod is running
crictl ps | grep apiserver
# (empty = API server not running)

# Check the static pod manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml
# Look for syntax errors or wrong flags

# Check API server logs
crictl logs $(crictl ps -a | grep apiserver | awk '{print $1}' | head -1)

# Check etcd connectivity (API server needs etcd)
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check certificate expiry
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
# notAfter= ... ← if expired, see Q166

# Check if port 6443 is listening
ss -tlnp | grep 6443
```

---

## Q160: Scheduler Not Scheduling Pods

```bash
# Check scheduler pod
kubectl get pods -n kube-system | grep scheduler
# kube-scheduler-master   0/1   Error   ← not healthy

# Check scheduler logs
kubectl logs kube-scheduler-master -n kube-system

# Check API server connectivity from scheduler
kubectl describe pod kube-scheduler-master -n kube-system

# Verify scheduler static pod manifest
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Force kubelet to restart by touching manifest
touch /etc/kubernetes/manifests/kube-scheduler.yaml
```

Common errors:
```
# Scheduler cannot reach API server
"error dialing backend: dial tcp 10.0.0.1:6443: connect: connection refused"

# Leader election issue
"leaderelection: error retrieving resource lock"
```

---

## Q161: Controller Manager Not Creating Replicas

```bash
# Check controller manager
kubectl get pods -n kube-system | grep controller-manager
kubectl logs kube-controller-manager-master -n kube-system | tail -20

# If replicaset stuck — check controller manager logs for this specific error
kubectl get replicaset -n <namespace>
kubectl describe replicaset <rs-name> -n <namespace>
# Events: "failed to create pod: ..."

# Check controller manager manifest
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

---

## Q162: Kubelet Not Registering with Cluster

```bash
# On the worker node
journalctl -u kubelet -n 50

# Common errors:
# "Unable to register node ... connection refused" → API server IP wrong
# "x509: certificate has expired" → kubelet cert expired

# Check bootstrap kubeconfig
cat /etc/kubernetes/kubelet.conf | grep server
# Ensure this matches the actual API server address

# Check kubelet service
systemctl status kubelet
systemctl restart kubelet

# On control plane: node should appear
kubectl get nodes
```

---

## Q163: CoreDNS Pods Not Running

```bash
# Check pod status
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                   READY   STATUS             RESTARTS
# coredns-abc123         0/1     CrashLoopBackOff   3

# Check logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Common issues:
# 1. ConfigMap syntax error
kubectl describe configmap coredns -n kube-system
kubectl edit configmap coredns -n kube-system  # fix Corefile

# 2. RBAC missing
kubectl get clusterrole system:coredns
kubectl get clusterrolebinding system:coredns

# 3. Resources: check if loop plugin causing CPU spike
kubectl top pod -n kube-system -l k8s-app=kube-dns

# Restart CoreDNS
kubectl rollout restart deployment/coredns -n kube-system
```

---

## Q164: kube-proxy Not Updating iptables

```bash
# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy | tail -20

# Check iptables rules exist for a service
kubectl get service web-service -n default
# CLUSTER-IP: 10.96.100.1

iptables -t nat -L KUBE-SERVICES | grep 10.96.100.1
# If empty: kube-proxy not writing rules

# Check kube-proxy ConfigMap
kubectl describe configmap kube-proxy -n kube-system

# Restart kube-proxy DaemonSet
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

---

## Q165: etcd Cluster Unhealthy

```bash
# Check etcd pod
kubectl get pods -n kube-system | grep etcd
kubectl logs etcd-master -n kube-system | tail -30

# Check cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# If corrupted — restore from backup (see Q19)
```

---

## Q166: Certificate Expiration

```bash
# Check all certificate expiry dates
kubeadm certs check-expiration
# CERTIFICATE           EXPIRES              RESIDUAL TIME
# admin.conf            Jan 15, 2024 10:00   EXPIRED ← fix this
# apiserver             Mar 15, 2025 10:00   364d

# Renew all certificates
kubeadm certs renew all
# [renew] certificate apiserver renewed
# [renew] certificate etcd-healthcheck-client renewed
# ...

# Restart control plane components (read new certs)
crictl ps | grep -E 'apiserver|scheduler|controller'
# Move and restore manifests to force restart:
for f in kube-apiserver kube-scheduler kube-controller-manager; do
  mv /etc/kubernetes/manifests/${f}.yaml /tmp/
done
sleep 10
for f in kube-apiserver kube-scheduler kube-controller-manager; do
  mv /tmp/${f}.yaml /etc/kubernetes/manifests/
done

# Update kubeconfig (admin cert also renewed)
cp /etc/kubernetes/admin.conf ~/.kube/config

# Verify
kubeadm certs check-expiration
kubectl get nodes
```
