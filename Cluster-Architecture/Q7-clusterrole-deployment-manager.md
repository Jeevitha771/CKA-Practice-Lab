# Q7. ClusterRole `deployment-manager` with Full Permissions

## Task
1. Create ClusterRole `deployment-manager` with full permissions on deployments and replicasets
2. Bind to user `admin@example.com`

---

## Solution

```bash
kubectl create clusterrole deployment-manager \
  --verb=get,list,watch,create,update,patch,delete \
  --resource=deployments,replicasets

kubectl create clusterrolebinding deployment-manager-binding \
  --clusterrole=deployment-manager \
  --user=admin@example.com
```

## Verify

```bash
kubectl auth can-i create deployments \
  --as=admin@example.com
# yes

kubectl auth can-i delete replicasets \
  --as=admin@example.com
# yes

kubectl auth can-i delete nodes \
  --as=admin@example.com
# no
```

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
- apiGroups: ["apps"]                        # deployments and replicasets are in "apps" group
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deployment-manager-binding
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

## Common API Groups

| Resource | apiGroup |
|---|---|
| pods, services, configmaps, secrets, serviceaccounts | `""` (core) |
| deployments, replicasets, daemonsets, statefulsets | `apps` |
| ingresses | `networking.k8s.io` |
| roles, rolebindings, clusterroles, clusterrolebindings | `rbac.authorization.k8s.io` |
| jobs, cronjobs | `batch` |
| horizontalpodautoscalers | `autoscaling` |
