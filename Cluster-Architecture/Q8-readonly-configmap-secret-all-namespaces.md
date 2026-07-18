# Q8. ServiceAccount Read-Only Access to ConfigMaps and Secrets Across All Namespaces

## Task
ServiceAccount `logger-sa` in namespace `logging` needs read-only access to ConfigMaps and Secrets across **all namespaces**. Create appropriate RBAC resources.

---

## Solution

```bash
kubectl create namespace logging
kubectl create serviceaccount logger-sa -n logging

# ClusterRole: read-only on configmaps and secrets
kubectl create clusterrole configmap-secret-reader \
  --verb=get,list,watch \
  --resource=configmaps,secrets

# ClusterRoleBinding: bind to the ServiceAccount
kubectl create clusterrolebinding configmap-secret-reader-binding \
  --clusterrole=configmap-secret-reader \
  --serviceaccount=logging:logger-sa
```

## Verify

```bash
# Can read ConfigMaps in any namespace
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:logging:logger-sa \
  -n default
# yes

kubectl auth can-i list secrets \
  --as=system:serviceaccount:logging:logger-sa \
  -n kube-system
# yes

# Cannot modify
kubectl auth can-i create configmaps \
  --as=system:serviceaccount:logging:logger-sa \
  -n default
# no

kubectl auth can-i delete secrets \
  --as=system:serviceaccount:logging:logger-sa \
  -n default
# no
```

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: configmap-secret-reader
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: configmap-secret-reader-binding
subjects:
- kind: ServiceAccount
  name: logger-sa
  namespace: logging
roleRef:
  kind: ClusterRole
  name: configmap-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## Security Note

> ⚠️ Granting read access to Secrets cluster-wide is a **high-privilege** operation.
> Secrets may contain passwords, tokens, and TLS keys.
> In production, prefer namespace-scoped RoleBindings where possible.
