# Q9. Role for Pods, Services, and Endpoints — Bound to ServiceAccount

## Task
In namespace `staging`:
1. Create a Role allowing get, list, watch on pods, services, and endpoints
2. Bind to ServiceAccount `app-viewer`

---

## Solution

```bash
kubectl create namespace staging
kubectl create serviceaccount app-viewer -n staging

kubectl create role resource-viewer \
  --verb=get,list,watch \
  --resource=pods,services,endpoints \
  -n staging

kubectl create rolebinding resource-viewer-binding \
  --role=resource-viewer \
  --serviceaccount=staging:app-viewer \
  -n staging
```

## Verify

```bash
for resource in pods services endpoints; do
  echo -n "list $resource: "
  kubectl auth can-i list $resource \
    --as=system:serviceaccount:staging:app-viewer \
    -n staging
done
# list pods: yes
# list services: yes
# list endpoints: yes

kubectl auth can-i create pods \
  --as=system:serviceaccount:staging:app-viewer \
  -n staging
# no
```

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: resource-viewer
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: resource-viewer-binding
  namespace: staging
subjects:
- kind: ServiceAccount
  name: app-viewer
  namespace: staging
roleRef:
  kind: Role
  name: resource-viewer
  apiGroup: rbac.authorization.k8s.io
```

## Notes

- `endpoints` is a core API resource (apiGroup: `""`)
- In Kubernetes 1.21+, `EndpointSlices` (`discovery.k8s.io`) are preferred over `endpoints`
- This Role is scoped to the `staging` namespace only — the ServiceAccount cannot see pods/services in other namespaces
