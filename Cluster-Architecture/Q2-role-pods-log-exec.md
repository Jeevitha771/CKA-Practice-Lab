# Q2. Role with pod/log and pod/exec Subresource Access

## Task
In namespace `dev`:
1. Create ServiceAccount `dev-user`
2. Create Role `pod-manager` with permissions: get, list, watch pods + access `pods/log` and `pods/exec` subresources
3. Create RoleBinding `dev-user-binding`

---

## Solution

```bash
kubectl create namespace dev

kubectl create serviceaccount dev-user -n dev
```

> ⚠️ `kubectl create role` cannot set subresources via flags — use YAML for subresources.

```yaml
# role-pod-manager.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]       # subresource for logs
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods/exec"]      # subresource for exec
  verbs: ["create"]             # exec requires "create" verb
```

```bash
kubectl apply -f role-pod-manager.yaml

kubectl create rolebinding dev-user-binding \
  --role=pod-manager \
  --serviceaccount=dev:dev-user \
  -n dev
```

## Verify

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:dev:dev-user -n dev
# yes

kubectl auth can-i get pods/log \
  --as=system:serviceaccount:dev:dev-user -n dev
# yes

kubectl auth can-i create pods/exec \
  --as=system:serviceaccount:dev:dev-user -n dev
# yes

kubectl auth can-i delete pods \
  --as=system:serviceaccount:dev:dev-user -n dev
# no
```

## Key Points

- `pods/log` → subresource for `kubectl logs` — verb: `get`
- `pods/exec` → subresource for `kubectl exec` — verb: `create`
- `pods/portforward` → subresource for `kubectl port-forward` — verb: `create`
- Subresources are listed separately in `resources:` — not as part of the parent resource rule
