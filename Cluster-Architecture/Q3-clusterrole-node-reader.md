# Q3. ClusterRole `node-reader` and ClusterRoleBinding for User

## Task
1. Create ClusterRole `node-reader` that can get, list, watch nodes cluster-wide
2. Create ClusterRoleBinding `node-reader-binding` for user `john@example.com`

---

## Solution

```bash
# Create the ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# Create the ClusterRoleBinding
kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=john@example.com
```

## Verify

```bash
kubectl get clusterrole node-reader
kubectl get clusterrolebinding node-reader-binding

# Test permissions as the user
kubectl auth can-i list nodes \
  --as=john@example.com
# Expected: yes

kubectl auth can-i delete nodes \
  --as=john@example.com
# Expected: no

kubectl auth can-i list pods \
  --as=john@example.com
# Expected: no (ClusterRole only covers nodes)
```

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: User
  name: john@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## Role vs ClusterRole

| Feature | Role | ClusterRole |
|---|---|---|
| Scope | Single namespace | Cluster-wide |
| Can grant access to nodes | ❌ (nodes are cluster-scoped) | ✅ |
| Can grant access to namespaced resources | ✅ | ✅ (all namespaces) |
| Used with RoleBinding | ✅ | ✅ (limits to one namespace) |
| Used with ClusterRoleBinding | ❌ | ✅ (cluster-wide) |
