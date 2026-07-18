# Q5. Role `secret-manager` Bound to a Group

## Task
In namespace `finance`:
1. Create Role `secret-manager` allowing create, update, delete on secrets
2. Bind it to group `finance-admins`

---

## Solution

```bash
kubectl create namespace finance

kubectl create role secret-manager \
  --verb=create,update,delete \
  --resource=secrets \
  -n finance

kubectl create rolebinding secret-manager-binding \
  --role=secret-manager \
  --group=finance-admins \
  -n finance
```

## Verify

```bash
kubectl get role secret-manager -n finance
kubectl get rolebinding secret-manager-binding -n finance

# Test as a user in the finance-admins group
kubectl auth can-i create secrets \
  --as=alice@example.com \
  --as-group=finance-admins \
  -n finance
# yes

kubectl auth can-i get secrets \
  --as=alice@example.com \
  --as-group=finance-admins \
  -n finance
# no  (only create/update/delete was granted)
```

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-manager
  namespace: finance
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-manager-binding
  namespace: finance
subjects:
- kind: Group                   # binding to a group — not a user or SA
  name: finance-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: secret-manager
  apiGroup: rbac.authorization.k8s.io
```

## Subject Types in RBAC

| Kind | Example | Notes |
|---|---|---|
| `User` | `john@example.com` | Individual user (from cert CN or OIDC) |
| `Group` | `finance-admins` | Group of users (from cert O or OIDC) |
| `ServiceAccount` | `app-sa` | Pod identity; requires `namespace` field |
