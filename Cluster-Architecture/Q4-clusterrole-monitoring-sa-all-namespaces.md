# Q4. Grant ServiceAccount Cluster-Wide Pod Access Across All Namespaces

## Task
Grant ServiceAccount `monitoring-sa` in namespace `monitoring` cluster-wide permissions to list and get pods in ALL namespaces using ClusterRole and ClusterRoleBinding.

---

## Solution

```bash
kubectl create namespace monitoring

kubectl create serviceaccount monitoring-sa -n monitoring

# ClusterRole — list/get pods across all namespaces
kubectl create clusterrole pod-lister \
  --verb=get,list \
  --resource=pods

# ClusterRoleBinding — bind to the ServiceAccount
kubectl create clusterrolebinding pod-lister-binding \
  --clusterrole=pod-lister \
  --serviceaccount=monitoring:monitoring-sa
```

## Verify

```bash
# Can list pods in default namespace
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:monitoring-sa \
  -n default
# yes

# Can list pods in kube-system
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:monitoring-sa \
  -n kube-system
# yes

# Cannot delete pods
kubectl auth can-i delete pods \
  --as=system:serviceaccount:monitoring:monitoring-sa \
  -n default
# no
```

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-lister
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-lister-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring         # namespace where SA lives
roleRef:
  kind: ClusterRole
  name: pod-lister
  apiGroup: rbac.authorization.k8s.io
```

## Key Insight: RoleBinding vs ClusterRoleBinding with ClusterRole

| Binding Type | Result |
|---|---|
| `ClusterRoleBinding` → `ClusterRole` | Access to resource **in all namespaces** |
| `RoleBinding` → `ClusterRole` | Access to resource **only in binding's namespace** |

> Use `ClusterRoleBinding` when the ServiceAccount needs cross-namespace access.
