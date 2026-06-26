# CKA Master Commands Cheat Sheet

> All kubectl and Linux commands, organized section-by-section.
> For the CKA exam — set your alias first: `alias k=kubectl`

---

## ⚡ Exam Setup (Run First)

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"    # k run nginx --image=nginx $do
export now="--force --grace-period=0"   # k delete pod nginx $now
source <(kubectl completion bash)
complete -F __start_kubectl k
```

---

## 1. DEPLOYMENTS

### Create & Configure

```bash
# Create deployment
kubectl create deployment web-app --image=nginx:1.21 --replicas=3 -n production

# Add labels to deployment (pod template)
kubectl label deployment web-app app=web tier=frontend env=production -n production

# Set resource requests and limits
kubectl set resources deployment web-app \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=200m,memory=256Mi \
  -n production

# Set rolling update strategy
kubectl patch deployment web-app -n production \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'

# Set minReadySeconds and progressDeadlineSeconds
kubectl patch deployment web-app \
  -p '{"spec":{"minReadySeconds":30,"progressDeadlineSeconds":600}}'

# Patch pod template labels
kubectl patch deployment web-app \
  -p '{"spec":{"template":{"metadata":{"labels":{"env":"production"}}}}}'
```

### Update Image

```bash
kubectl set image deployment/web-app nginx=nginx:1.22 --record
kubectl set image deployment/web-app nginx=nginx:1.22 -n production
```

### Rollout Management

```bash
kubectl rollout status deployment/web-app
kubectl rollout history deployment/web-app
kubectl rollout history deployment/web-app --revision=2
kubectl rollout undo deployment/web-app
kubectl rollout undo deployment/web-app --to-revision=1
kubectl rollout pause deployment web-app
kubectl rollout resume deployment web-app
kubectl rollout restart deployment web-app
```

### Scale

```bash
kubectl scale deployment web-app --replicas=5
kubectl scale deployment web-app --replicas=8 -n production
kubectl scale statefulset database --replicas=5
```

### Canary

```bash
kubectl create deployment web-app-canary --image=nginx:1.22 --replicas=1
kubectl label deployment web-app-canary app=web tier=frontend
kubectl scale deployment web-app --replicas=0          # shift all traffic to canary
kubectl scale deployment web-app-canary --replicas=5
```

### Pause → Batch Changes → Resume

```bash
kubectl rollout pause deployment web-app
kubectl set image deployment/web-app nginx=nginx:1.23
kubectl set env deployment/web-app ENV=prod
kubectl set resources deployment web-app --limits=cpu=500m,memory=512Mi --requests=cpu=200m,memory=256Mi
kubectl rollout resume deployment web-app
kubectl rollout history deployment web-app    # should show only 1 new revision
```

### Revision History Limit

```bash
kubectl patch deployment web-app -p '{"spec":{"revisionHistoryLimit":5}}'

# Trigger 8 updates via env var (forces pod template change = real rollout)
for i in {1..8}; do
  kubectl set env deploy web-updater -n update-test UPDATE_ID=$i
  sleep 1
done

# Check revisions kept
kubectl rollout history deployment/web-updater -n update-test
kubectl get rs -n update-test > /opt/outputs/web-updater-rs.txt
```

### Inspect

```bash
kubectl get deployment web-app -o wide
kubectl describe deployment web-app
kubectl get pods -l app=web -o wide
kubectl get pods -w
kubectl explain deployment.spec | grep -i seconds
```

---

## 2. CONFIGMAPS & SECRETS

### Create ConfigMap

```bash
# From literals
kubectl create configmap app-config \
  --from-literal=database_url='postgres://db:5432/mydb' \
  --from-literal=log_level='INFO' \
  -n production

# From file
kubectl create configmap app-config --from-file=/opt/config/app.properties

# From env file
kubectl create configmap app-config --from-env-file=/opt/config/app.env

# Combined
kubectl create configmap app-config \
  --from-literal=database_url='postgres://db:5432/mydb' \
  --from-literal=log_level='INFO' \
  --from-file=/opt/config/app.properties \
  --from-env-file=/opt/config/app.env \
  -n production
```

### Create Secret

```bash
# Generic (Opaque)
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret

# TLS secret
kubectl create secret tls my-tls-secret \
  --cert=/opt/certs/server.crt \
  --key=/opt/certs/server.key

# Generate TLS certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /opt/certs/server.key \
  -out /opt/certs/server.crt \
  -subj "/CN=myapp"
```

### Inspect

```bash
kubectl get configmap app-config -o yaml -n production
kubectl describe configmap app-config -n production
kubectl get secret db-credentials -o yaml
kubectl describe secret db-credentials
```

### Inject ConfigMap into Deployment

```bash
kubectl set env --from=configmap/app-config deployment/web-app
```

### Update & Reload

```bash
kubectl edit configmap app-config
kubectl apply -f configmap.yaml
kubectl rollout restart deployment web-app    # force pods to reload env vars
kubectl exec -it <pod> -- printenv            # verify inside pod
kubectl exec -it <pod> -- printenv APP_MODE
```

### Immutable ConfigMap/Secret — Delete & Recreate

```bash
kubectl delete configmap app-config
kubectl create configmap app-config --from-literal=mode=dev
kubectl replace --force -f /tmp/kubectl-edit-*.yaml
```

---

## 3. SCALING & AUTOSCALING

### HPA (HorizontalPodAutoscaler)

```bash
# Install metrics-server (if missing)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Fix TLS on kubeadm clusters
kubectl edit deployment metrics-server -n kube-system
# Add: - --kubelet-insecure-tls

# Verify metrics working
kubectl top nodes
kubectl top pods
kubectl top pods -A

# Create HPA imperatively
kubectl autoscale deployment web-frontend --cpu-percent=60 --min=2 --max=10

# Watch autoscaling
kubectl get hpa
kubectl get hpa -w
kubectl describe hpa web-frontend-hpa
```

### Generate Load

```bash
kubectl run load-generator --rm -it --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://web-frontend; done"
```

### VPA (VerticalPodAutoscaler)

```bash
# Check if VPA CRD exists
kubectl get crd | grep verticalpodautoscaler

# Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler && ./hack/vpa-up.sh

# Check VPA components
kubectl get pods -n kube-system | grep vpa

kubectl get vpa
kubectl describe vpa webapp-vpa
```

---

## 4. PROBES & HEALTH CHECKS

```bash
kubectl explain pod.spec.containers.livenessProbe
kubectl explain pod.spec.containers.readinessProbe
kubectl explain pod.spec.containers.startupProbe

kubectl describe pod <pod> | grep -A20 "Liveness\|Readiness\|Startup"
kubectl get pods -w                      # watch restart count
kubectl describe pod liveness-demo       # see probe failure events
```

---

## 5. DAEMONSETS

```bash
kubectl create daemonset node-monitor --image=prometheus/node-exporter \
  --dry-run=client -o yaml > ds.yaml   # generate base then add tolerations

kubectl get daemonset -n kube-system
kubectl get pods -l app=node-monitor -o wide    # verify 1 pod per node
kubectl describe daemonset node-monitor
```

---

## 6. JOBS & CRONJOBS

```bash
# Create Job imperatively
kubectl create job batch-processor --image=busybox \
  --dry-run=client -o yaml \
  -- /bin/sh -c 'echo "Processing $HOSTNAME" && sleep 10' > job.yaml

# Monitor Job
kubectl get job batch-processor -w
kubectl get pods -l job-name=batch-processor
kubectl logs -l job-name=batch-processor
kubectl describe job batch-processor

# Create CronJob
kubectl create cronjob pvc-backup --image=busybox \
  --schedule="0 2 * * *" \
  --dry-run=client -o yaml > cronjob.yaml

# Manually trigger CronJob
kubectl create job --from=cronjob/pvc-backup-cronjob manual-test

kubectl get cronjob
kubectl get jobs
```

---

## 7. RESOURCE MANAGEMENT

### ResourceQuota

```bash
# Create namespace with quota
kubectl create namespace limited

kubectl create quota limited-quota \
  --hard=requests.cpu=4,requests.memory=8Gi,limits.cpu=8,limits.memory=16Gi,\
count/pods=20,count/services=10,count/persistentvolumeclaims=15 \
  --namespace=limited

kubectl get quota -n limited
kubectl describe resourcequota limited-quota -n limited
```

### LimitRange

```bash
kubectl get limitrange -n dev
kubectl describe limitrange dev-container-limits -n dev

# Test auto-injection of defaults
kubectl run test-pod --image=nginx -n dev
kubectl get pod test-pod -n dev -o yaml | grep -A10 resources
```

### Debug Pending / Insufficient CPU

```bash
kubectl get pods
kubectl describe pod <pod>              # look for FailedScheduling events
kubectl describe nodes | grep -A10 "Allocated resources"
kubectl get quota -n <namespace>
kubectl get limitrange -n <namespace>

# Fix: reduce requests
kubectl delete pod cpu-hog-pod
# Edit YAML to change cpu: "100" → cpu: "100m"
kubectl apply -f fixed-pod.yaml
```

### Capacity Math

```bash
# Check node allocatable resources
kubectl describe node <node-name> | grep -A8 "Allocatable"
kubectl describe node <node-name> | grep -A10 "Allocated resources"

# Formula:
# Allocatable = Capacity - System Reserved
# Available   = Allocatable - Existing Pod REQUESTS
# If Available >= New Pod REQUEST → schedules
```

---

## 8. SCHEDULING

### NodeSelector / Labels

```bash
kubectl label node node1 disktype=ssd
kubectl label node node1 topology.kubernetes.io/zone=us-east-1
kubectl label node node1 disktype-          # remove label
kubectl get nodes --show-labels
```

### Taints and Tolerations

```bash
# Apply taint
kubectl taint nodes node1 special=true:NoSchedule
kubectl taint nodes node1 special=true:NoExecute

# Remove taint (append -)
kubectl taint nodes node1 special=true:NoSchedule-

# Verify taints
kubectl describe node node1 | grep Taints

# Simulate node crash (apply NoExecute manually)
kubectl taint nodes node1 node.kubernetes.io/not-ready:NoExecute
```

### PriorityClass

```bash
kubectl get priorityclass
kubectl describe priorityclass high-priority
kubectl get pod nginx-high-priority -o yaml | grep priorityClassName
```

### Affinity Debug

```bash
kubectl get pods -o wide                    # check which node pod landed on
kubectl describe pod <pod> | grep -A20 Events
```

---

## 9. MANIFEST MANAGEMENT

### dry-run + YAML generation

```bash
# Generate Pod YAML
kubectl run nginx --image=nginx:1.21 --port=80 \
  --env="ENV=production" --labels="app=web,tier=frontend" \
  --dry-run=client -o yaml > pod.yaml

# Generate Deployment YAML
kubectl create deployment api --image=api:v1 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Generate Service YAML
kubectl create service clusterip api --tcp=8080:8080 \
  --dry-run=client -o yaml > service.yaml

# Generate ConfigMap from directory
kubectl create configmap api-config --from-file=/opt/config \
  --dry-run=client -o yaml > configmap.yaml

# Expose deployment
kubectl expose deployment web-app --name=web-svc --port=80 --target-port=80 \
  --dry-run=client -o yaml > svc.yaml
```

### Validate

```bash
kubectl apply -f manifest.yaml --dry-run=client     # local syntax check only
kubectl apply -f manifest.yaml --dry-run=server     # full API server validation
kubectl diff -f manifest.yaml                       # preview what will change
```

### Apply & Delete

```bash
kubectl apply -f manifest.yaml
kubectl apply -f directory/
kubectl delete -f manifest.yaml
kubectl replace --force -f manifest.yaml            # delete + recreate
```

### Kustomize

```bash
kubectl kustomize overlays/dev          # preview (no apply)
kubectl kustomize overlays/prod
kubectl apply -k overlays/dev           # apply dev environment
kubectl delete -k overlays/dev
```

---

## 10. NETWORKING

### CNI & Network Config

```bash
# Identify CNI plugin
kubectl get pods -n kube-system | grep -E 'calico|flannel|cilium|weave'
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist

# Find Pod CIDR
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr

# Find Service CIDR
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range

# Find Cluster DNS IP
kubectl get svc kube-dns -n kube-system

# Per-node Pod CIDR assignments
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

### IP Forwarding & br_netfilter

```bash
# Check
sysctl net.ipv4.ip_forward                  # must be 1
lsmod | grep br_netfilter                   # must show output

# Fix temporarily
sysctl -w net.ipv4.ip_forward=1
modprobe br_netfilter

# Fix permanently
cat <<EOF >> /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system

cat <<EOF >> /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF
```

### Pod Connectivity Debug

```bash
# Create test pods
kubectl run source-pod --image=busybox:1.28 --command -- sleep 3600
kubectl run dest-pod --image=nginx --labels="app=dest"
kubectl expose pod dest-pod --name=dest-service --port=80

# Get pod IP
kubectl get pod dest-pod -o wide

# Test IP connectivity (bypasses DNS)
kubectl exec -it source-pod -- ping <DEST_POD_IP>

# Test DNS
kubectl exec -it source-pod -- nslookup dest-service
kubectl exec -it source-pod -- nslookup dest-service.default.svc.cluster.local

# Check /etc/resolv.conf
kubectl exec -it source-pod -- cat /etc/resolv.conf

# Check node routes
ip route
ip link show type veth
brctl show

# Check iptables
iptables-save | grep <pod-ip>
iptables-save | grep DROP

# Cleanup
kubectl delete pod source-pod dest-pod
kubectl delete svc dest-service
```

### CNI Pod Logs

```bash
kubectl logs -n kube-system <cni-pod-name>
kubectl describe pod -n kube-system <cni-pod-name>
```

### NetworkPolicy

```bash
kubectl get networkpolicies --all-namespaces
kubectl get netpol -n production
kubectl describe netpol default-deny-all -n production
```

---

## 11. STORAGE

### PersistentVolume

```bash
kubectl get pv
kubectl get pv | grep Available
kubectl get pv | grep Bound
kubectl get pv | grep Released
kubectl describe pv <pv-name>

# Reclaim Released PV (fastest)
kubectl patch pv pv-data -p '{"spec":{"claimRef": null}}'

# Change reclaim policy
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl edit pv <pv-name>
kubectl get pv <pv-name> -o yaml | grep persistentVolumeReclaimPolicy
```

### PersistentVolumeClaim

```bash
kubectl get pvc
kubectl get pvc -n production
kubectl describe pvc data-claim

# Debug Pending PVC
kubectl describe pvc <name>             # read Events section for error
kubectl get sc                          # check storageclass exists
kubectl get pv --show-labels            # check PV labels match PVC selector

# Expand PVC
kubectl edit pvc data-claim             # change spec.resources.requests.storage
kubectl get pvc data-claim -o jsonpath='{.spec.resources.requests.storage}'

# Check expansion support
kubectl get sc fast-storage -o yaml | grep allowVolumeExpansion
kubectl patch sc fast-storage -p '{"allowVolumeExpansion": true}'
```

### StorageClass

```bash
kubectl get sc
kubectl describe sc fast-storage
```

### StatefulSet + PVCs

```bash
kubectl get statefulset
kubectl describe statefulset database
kubectl get pvc -l app=database         # verify unique PVC per pod
kubectl get pods -l app=database -o wide
```

### Verify Persistence

```bash
# Write data
kubectl exec -it pvc-pod -- sh -c 'echo "hello cka" > /data/test.txt'
kubectl exec -it pvc-pod -- cat /data/test.txt

# Delete and recreate pod
kubectl delete pod pvc-pod
kubectl apply -f pod.yaml

# Verify data survived
kubectl exec -it pvc-pod -- cat /data/test.txt
```

### Volumes in Pod

```bash
# Check mounted volumes
kubectl describe pod <pod> | grep -A20 "Volumes\|Mounts"
kubectl exec -it <pod> -- df -h /data
kubectl exec -it <pod> -- ls -la /config
kubectl exec -it <pod> -- ls -la /secrets
kubectl exec -it <pod> -- ls -la /projected-data
```

### Job for PVC Backup

```bash
kubectl apply -f backup-job.yaml
kubectl get job pvc-backup-job
kubectl logs -l job-name=pvc-backup-job
kubectl describe job pvc-backup-job
```

---

## 12. DEBUGGING & TROUBLESHOOTING

### Pod Lifecycle States

```bash
kubectl get pods                         # check STATUS column
kubectl get pods -A                      # all namespaces
kubectl get pods -o wide                 # with node and IP
kubectl get pods -w                      # watch live
kubectl get pods --show-labels

kubectl describe pod <pod>               # full detail + events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n <namespace> --field-selector reason=Failed
```

### Logs

```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>        # specific container
kubectl logs <pod> --previous            # previous crashed container
kubectl logs <pod> -f                    # stream/follow
kubectl logs -l app=web                  # all pods with label
kubectl logs <pod> | grep ERROR
kubectl logs -n kube-system <cni-pod>
```

### Exec into Pod

```bash
kubectl exec -it <pod> -- /bin/sh
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -c <container> -- /bin/sh
kubectl exec <pod> -- env                # print env vars
kubectl exec <pod> -- printenv DB_URL
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- cat /data/test.txt
kubectl exec <pod> -- ls -la /etc/db-credentials/
kubectl exec <pod> -- df -h /data
```

### Node Debugging

```bash
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl describe node <node-name> | grep -A10 "Conditions"
kubectl describe node <node-name> | grep -A10 "Allocated resources"
kubectl top nodes
kubectl cordon <node>                    # prevent new pods
kubectl uncordon <node>                  # re-enable scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

### RBAC / Permissions Check

```bash
kubectl auth can-i create pods -n production
kubectl auth can-i create deployments --as=system:serviceaccount:production:app-sa
kubectl auth can-i --list --as=developer@example.com -n dev
```

### Resource Usage

```bash
kubectl top nodes
kubectl top pods
kubectl top pods -A
kubectl top pods -n production --sort-by=cpu
kubectl top pods -n production --sort-by=memory
kubectl top pod <pod> --containers         # per-container usage
```

### Service / Endpoint Debug

```bash
kubectl get svc
kubectl get svc -n production
kubectl describe svc backend-service
kubectl get endpoints backend-service
kubectl get endpoints -n production        # verify pods registered

# Test DNS from a temporary pod
kubectl run test --image=busybox:1.28 -it --rm --restart=Never \
  -- nslookup backend-service
```

### CrashLoopBackOff / OOMKilled

```bash
kubectl describe pod <pod> | grep -i "last state\|reason\|exit code"
kubectl logs <pod> --previous
# OOMKilled → increase memory limits
# Exit Code 1 → app crash → check app logs
# Exit Code 137 → OOM killed by kernel
```

### Delete Quickly

```bash
kubectl delete pod <pod> --force --grace-period=0
kubectl delete pod <pod> --now
```

---

## 13. CLUSTER COMPONENTS (etcd / kubeadm)

### etcd Backup & Restore

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Health check
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Reclaim Released PV
kubectl patch pv <name> -p '{"spec":{"claimRef": null}}'
```

### kubeadm

```bash
# Initialize cluster
kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version=v1.28.0 \
  --control-plane-endpoint=k8s-master.local

# Configure kubectl after init
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Generate join command
kubeadm token create --print-join-command

# Check upgrade plan
kubeadm upgrade plan

# Upgrade control plane
kubeadm upgrade apply v1.28.0

# Drain + upgrade worker node
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon worker-1

# Certificate renewal
kubeadm certs check-expiration
kubeadm certs renew all
```

### Node Prereqs Check

```bash
# Swap
swapon --show                            # must be empty
swapoff -a

# Kernel modules
lsmod | grep br_netfilter
lsmod | grep overlay
modprobe br_netfilter
modprobe overlay

# sysctl
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables

# Container runtime
systemctl status containerd
crictl info
crictl ps
```

---

## 14. SYSTEMD & NODE-LEVEL

```bash
# kubelet
systemctl status kubelet
systemctl restart kubelet
journalctl -u kubelet -f
journalctl -u kubelet --since "10 minutes ago"

# containerd
systemctl status containerd
crictl ps
crictl images
crictl pull nginx:alpine
crictl logs <container-id>

# Node networking
ip route
ip link
ip addr
ip link show type veth
brctl show
sysctl net.ipv4.ip_forward
conntrack -L
iptables-save | grep -v "^#" | head -50
ipvsadm -Ln                              # if IPVS mode
```

---

## 15. QUICK REFERENCE — Common Patterns

### Generate YAML for any resource

```bash
kubectl run <name> --image=<image> --dry-run=client -o yaml
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml
kubectl create configmap <name> --from-literal=k=v --dry-run=client -o yaml
kubectl create secret generic <name> --from-literal=k=v --dry-run=client -o yaml
kubectl create job <name> --image=<image> --dry-run=client -o yaml
kubectl create cronjob <name> --image=<image> --schedule="0 2 * * *" --dry-run=client -o yaml
kubectl expose deployment <name> --port=80 --dry-run=client -o yaml
```

### Force delete

```bash
kubectl delete pod <pod> --force --grace-period=0
```

### Get specific field

```bash
kubectl get pod <pod> -o jsonpath='{.status.podIP}'
kubectl get node <node> -o jsonpath='{.spec.podCIDR}'
kubectl get pvc <pvc> -o jsonpath='{.spec.resources.requests.storage}'
kubectl get deployment <name> -o jsonpath='{.spec.replicas}'
```

### Watch

```bash
kubectl get pods -w
kubectl get hpa -w
kubectl get deployment -w
kubectl rollout status deployment/web-app
```

### All namespaces

```bash
kubectl get pods -A
kubectl get svc -A
kubectl get pvc -A
kubectl get netpol -A
kubectl get quota -A
kubectl get events -A --sort-by='.lastTimestamp'
```

### Explain fields

```bash
kubectl explain deployment.spec
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.affinity
kubectl explain pod.spec.tolerations
kubectl explain networkpolicy.spec
kubectl explain pvc.spec
kubectl explain statefulset.spec.updateStrategy
```

---

## 16. CRON SCHEDULE REFERENCE

```
┌─────── Minute   (0-59)
│ ┌───── Hour     (0-23)
│ │ ┌─── Day      (1-31)
│ │ │ ┌─ Month    (1-12)
│ │ │ │ ┌ Weekday (0-7, 0=Sun)
│ │ │ │ │
* * * * *

0 2 * * *     →  2:00 AM daily
0 */6 * * *   →  Every 6 hours
0 0 * * 0     →  Every Sunday midnight
*/5 * * * *   →  Every 5 minutes
```

---

## 17. DNS NAMES REFERENCE

```
# Service DNS (same namespace)
<service-name>
<service-name>.<namespace>
<service-name>.<namespace>.svc
<service-name>.<namespace>.svc.cluster.local

# StatefulSet Pod DNS (headless service)
<pod-name>.<service-name>.<namespace>.svc.cluster.local
database-0.database-svc.default.svc.cluster.local
database-1.database-svc.default.svc.cluster.local

# Examples
backend.default.svc.cluster.local
mysql.production.svc.cluster.local
redis-0.redis-headless.cache.svc.cluster.local
```

---

## 18. RESOURCE UNIT CONVERSIONS

```
CPU:
  1 CPU   = 1000m (millicores)
  0.5 CPU = 500m
  0.1 CPU = 100m

Memory:
  1Ki = 1024 bytes
  1Mi = 1024 Ki  = 1,048,576 bytes
  1Gi = 1024 Mi  = 1,073,741,824 bytes

  1G  = 1000 MB  (SI)
  1Gi = 1024 MiB (binary)

Scheduling formula:
  Allocatable = Capacity - System Reserved
  Available   = Allocatable - Existing Pod REQUESTS
  Schedules?  = Available >= New Pod Request (CPU AND Memory both)
```
