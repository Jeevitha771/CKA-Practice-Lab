# Q10. Verify and Enforce Create-but-Not-Delete Pods for a User

## Task
Verify user `developer@example.com` can create but NOT delete pods in namespace `dev`. Create necessary RBAC to enforce this.

---

## Phase 1: Check Current State

```bash
kubectl auth can-i create pods \
  --as=developer@example.com -n dev
# (may be yes or no depending on existing RBAC)

kubectl auth can-i delete pods \
  --as=developer@example.com -n dev
# should be no after enforcement
```

---

## Phase 2: Create Targeted RBAC

```bash
kubectl create namespace dev

# Allow: create + basic read on pods
kubectl create role pod-creator \
  --verb=create,get,list,watch \
  --resource=pods \
  -n dev

kubectl create rolebinding pod-creator-binding \
  --role=pod-creator \
  --user=developer@example.com \
  -n dev
```

> ⚠️ Do NOT include `delete` or `deletecollection` in the verbs — that enforces the restriction.

---

## Phase 3: Verify Enforcement

```bash
kubectl auth can-i create pods \
  --as=developer@example.com -n dev
# yes ✅

kubectl auth can-i delete pods \
  --as=developer@example.com -n dev
# no ✅

kubectl auth can-i list pods \
  --as=developer@example.com -n dev
# yes ✅
```

---

## Full Permissions Audit

```bash
kubectl auth can-i --list \
  --as=developer@example.com \
  -n dev
```

Sample output:
```
Resources  Non-Resource URLs  Resource Names  Verbs
pods       []                 []              [create get list watch]
```

---

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-creator
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "get", "list", "watch"]   # note: no "delete"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-creator-binding
  namespace: dev
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-creator
  apiGroup: rbac.authorization.k8s.io
```

## Key Points

- RBAC is **deny-by-default** — only explicitly listed verbs are allowed
- There is no "deny" verb in RBAC — simply omit the verb you want to block
- If the user also has `ClusterRoleBindings` or belongs to groups with more permissions, those take effect — RBAC is **additive**
