# Q72. Implement Pod Affinity — Prefer SSD Nodes, Require Same Zone as Backend, Avoid Co-located Frontend Pods

## Task
Implement pod affinity:
- **Prefer** nodes with label `disktype=ssd`
- **Require** same zone as `backend` pods
- **Avoid** multiple `frontend` pods on the same node

## Setup Labels

```bash
kubectl label node node1 topology.kubernetes.io/zone=us-east-1
kubectl label node node2 topology.kubernetes.io/zone=us-east-1
kubectl label node node1 disktype=ssd        # Only node1 gets SSD label
```

## Backend Pod (must exist first — frontend uses affinity to co-locate)

```yaml
# backend.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
  labels:
    app: backend
spec:
  containers:
  - name: redis
    image: redis:alpine
```

## Frontend Pod with All 3 Affinity Rules

```yaml
# frontend.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod-1
  labels:
    app: frontend
spec:
  affinity:

    # 1. PREFER nodes with disktype=ssd (soft rule — tries but doesn't require)
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd

    # 2. REQUIRE same zone as backend pods (hard rule — must be satisfied)
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - backend
        topologyKey: topology.kubernetes.io/zone

    # 3. AVOID multiple frontend pods on same node (anti-affinity — hard rule)
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - frontend
        topologyKey: kubernetes.io/hostname

  containers:
  - name: nginx
    image: nginx:alpine
```

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
kubectl get pods -o wide
# frontend-pod-1 should land on same zone node as backend, preferably node1 (has SSD)

# Try second frontend — should go to different node
kubectl apply -f frontend-pod-2.yaml
kubectl get pods -o wide
```

## Affinity Rule Types

| Rule | Keyword | Behaviour |
|---|---|---|
| Hard node | `requiredDuringSchedulingIgnoredDuringExecution` | Pod only schedules if condition met |
| Soft node | `preferredDuringSchedulingIgnoredDuringExecution` | Pod prefers but not required |
| Pod affinity | `podAffinity` | Schedule near pods matching selector |
| Pod anti-affinity | `podAntiAffinity` | Avoid scheduling near pods matching selector |

## topologyKey Reference

| topologyKey | Scope |
|---|---|
| `kubernetes.io/hostname` | Same node |
| `topology.kubernetes.io/zone` | Same availability zone |
| `topology.kubernetes.io/region` | Same region |

## Cleanup

```bash
kubectl delete pod backend-pod frontend-pod-1
kubectl label node node1 disktype-
```

## Reference
https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
