# Q1. Create ServiceAccount, Role `pod-reader`, and RoleBinding

## Task
In namespace `production`:
1. Create ServiceAccount `app-sa`
2. Create Role `pod-reader` that allows get, watch, list on pods
3. Bind the role to the ServiceAccount

---

## Solution

```bash
# Step 1: Create the namespace (if it doesn't exist)
kubectl create namespace production

# Step 2: Create the ServiceAccount
kubectl create serviceaccount app-sa -n production

# Step 3: Create the Role
kubectl create role pod-reader \
  --verb=get,watch,list \
  --resource=pods \
  -n production

# Step 4: Create the RoleBinding
kubectl create rolebinding pod-reader-binding \
  --role=pod-reader \
  --serviceaccount=production:app-sa \
  -n production
```

## Verify

```bash
# Check resources exist
kubectl get serviceaccount app-sa -n production
kubectl get role pod-reader -n production
kubectl get rolebinding pod-reader-binding -n production

# Confirm permissions
kubectl auth can-i list pods \
  --as=system:serviceaccount:production:app-sa \
  -n production
# Expected: yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:production:app-sa \
  -n production
# Expected: no
```

## Equivalent YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Key Points

| Resource | Scope | Purpose |
|---|---|---|
| ServiceAccount | Namespace | Identity for pods/processes |
| Role | Namespace | Defines allowed verbs on resources |
| RoleBinding | Namespace | Connects subject to a Role |
