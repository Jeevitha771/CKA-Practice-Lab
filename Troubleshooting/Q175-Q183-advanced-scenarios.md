# Q175–Q183. Advanced Troubleshooting Scenarios

## Q175: Full Application Stack Debug
## Q176: Post-Upgrade Application Failures
## Q177: Node High Load
## Q178: Complete Monitoring Solution
## Q179: Secure Namespace
## Q180: Migrate Stateful Application Without Data Loss
## Q181: Certificate Expiration Debug
## Q182: Service Mesh (Istio) Troubleshooting
## Q183: Disaster Recovery Drill

---

## Q175: Full Application Stack Debug (Frontend/Backend/Database)

```bash
# Systematic check: work from bottom (database) to top (frontend)

# 1. Database pods
kubectl get pods -n production -l tier=database
kubectl logs -n production -l tier=database | tail -20
kubectl get endpoints database-service -n production

# 2. Backend pods
kubectl get pods -n production -l tier=backend
kubectl exec backend-pod -n production -- \
  nslookup database-service.production.svc.cluster.local

# 3. Service → Endpoints chain
kubectl get svc,endpoints -n production

# 4. Ingress → Frontend chain
kubectl describe ingress -n production
kubectl get pods -n ingress-nginx

# 5. ConfigMaps / Secrets
kubectl get cm,secret -n production

# 6. NetworkPolicies
kubectl get networkpolicy -n production
kubectl describe networkpolicy -n production

# 7. Persistent storage
kubectl get pvc -n production
# All should be Bound
```

---

## Q176: Post-Upgrade Application Failures

```bash
# 1. Check deprecated API versions
kubectl api-resources
# Use kubectl-convert or check for deprecated resources

# Find resources using old API versions
kubectl get all -n production -o yaml | grep "apiVersion" | sort | uniq -c

# 2. Check RBAC changes
kubectl auth can-i --list -n production --as=system:serviceaccount:production:app-sa

# 3. Check changed default behaviors
kubectl get events -n kube-system | grep Warning

# 4. CNI plugin compatibility
kubectl get pods -n kube-system | grep -E 'calico|flannel'
# May need CNI upgrade after cluster upgrade

# 5. Inspect pods failing after upgrade
kubectl get pods -A | grep -v Running
kubectl describe pod <failing-pod> | grep -A 5 "Events:"
```

---

## Q177: Node High Load

```bash
# Identify which pods are causing high load
kubectl top pods -A --sort-by=cpu | head -10
kubectl top pods -A --sort-by=memory | head -10

# Implement resource limits
kubectl set resources deployment/heavy-app \
  --limits=cpu=500m,memory=512Mi \
  --requests=cpu=100m,memory=128Mi

# Priority classes — ensure critical pods aren't evicted
kubectl create priorityclass high-priority \
  --value=1000 \
  --global-default=false \
  --description="High priority workloads"

# Node affinity — spread load
kubectl patch deployment heavy-app -p '
{
  "spec": {
    "template": {
      "spec": {
        "affinity": {
          "podAntiAffinity": {
            "preferredDuringSchedulingIgnoredDuringExecution": [{
              "weight": 100,
              "podAffinityTerm": {
                "topologyKey": "kubernetes.io/hostname",
                "labelSelector": {
                  "matchLabels": {"app": "heavy-app"}
                }
              }
            }]
          }
        }
      }
    }
  }
}'

# Eviction policies via node conditions
kubectl describe node worker-1 | grep -A 5 "Conditions:"
```

---

## Q178: Complete Monitoring Solution

```bash
# 1. Deploy metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. Resource Quotas per namespace
kubectl create namespace production
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
EOF

# 3. HPA
kubectl autoscale deployment web-app \
  --cpu-percent=70 \
  --min=2 \
  --max=10 \
  -n production

# 4. Verify monitoring
kubectl top nodes
kubectl top pods -n production
kubectl describe hpa web-app -n production
```

---

## Q179: Secure Namespace

```bash
kubectl create namespace secure-ns

# 1. Default-deny NetworkPolicy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF

# 2. Pod Security Standards (Restricted)
kubectl label namespace secure-ns \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# 3. RBAC — least privilege
kubectl create role app-role \
  --verb=get,list,watch \
  --resource=pods,configmaps \
  -n secure-ns

# 4. ResourceQuota
kubectl create resourcequota secure-quota \
  --hard=pods=10,requests.cpu=2,limits.cpu=4 \
  -n secure-ns
```

---

## Q180: Migrate Stateful Application Without Data Loss

```bash
# 1. Cordon source node (no new pods scheduled)
kubectl cordon worker-1

# 2. Drain safely (moves pods, respects PodDisruptionBudgets)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# 3. Verify PVs reattached on new node
kubectl get pods -n production -o wide
# Pods should be running on other nodes

kubectl get pvc -n production
# All Bound ← PVs followed the pods

# 4. Validate application state
kubectl exec -n production stateful-pod-0 -- \
  psql -U postgres -c "SELECT count(*) FROM users;"

# 5. Uncordon when ready to use worker-1 again
kubectl uncordon worker-1
```

---

## Q181: Certificate Expiration Debug

```bash
# Full certificate expiry check
kubeadm certs check-expiration

# Check API server cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Check kubelet cert
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Check service account key
openssl x509 -in /etc/kubernetes/pki/sa.pub -noout -dates 2>/dev/null || \
  echo "SA key doesn't expire (RSA key, not cert)"

# Renew all
kubeadm certs renew all

# Restart affected components
systemctl restart kubelet
# Restart control plane via manifest touch:
for f in /etc/kubernetes/manifests/*.yaml; do touch $f; done
```

---

## Q183: Disaster Recovery Drill

```bash
# Step 1: Record current state
kubectl get all -A > /tmp/cluster-state-before.txt

# Step 2: Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /opt/dr-drill.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

START=$(date +%s)

# Step 3: Simulate failure — delete etcd data
systemctl stop kubelet
rm -rf /var/lib/etcd/*

# Step 4: Restore etcd (see Q19 for full steps)
ETCDCTL_API=3 etcdctl snapshot restore /opt/dr-drill.db \
  --data-dir=/var/lib/etcd

# Step 5: Restart and verify
systemctl start kubelet
sleep 60
kubectl get nodes
kubectl get all -A > /tmp/cluster-state-after.txt
diff /tmp/cluster-state-before.txt /tmp/cluster-state-after.txt

END=$(date +%s)
echo "RTO (Recovery Time Objective): $((END - START)) seconds"
```
