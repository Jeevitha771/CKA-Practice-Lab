# Q73. Taints and Tolerations — NoSchedule, NoExecute, tolerationSeconds

## Task
Apply taint to a node, deploy pods with and without tolerations, demonstrate NoSchedule and NoExecute effects.

## Concepts

- **Taint** = Repellent on a Node. Repels pods without matching toleration.
- **Toleration** = Immunity on a Pod. Allows it to land on tainted nodes.
- Toleration does NOT force a pod to go there — it only permits it (use Affinity to force).

## Taint Effects

| Effect | Behaviour |
|---|---|
| `NoSchedule` | New pods without toleration cannot be placed on the node. Running pods stay. |
| `NoExecute` | New pods blocked AND existing pods without toleration are evicted immediately. |
| `PreferNoSchedule` | Soft version of NoSchedule — scheduler tries to avoid but not required. |

## Setup

```bash
# Apply a custom taint to a node
kubectl taint nodes <node1-name> special=true:NoSchedule

# Verify
kubectl describe node <node1-name> | grep Taints
```

## Standard Pod (no toleration — will be blocked)

```yaml
# standard-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: standard-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
```

```bash
kubectl apply -f standard-pod.yaml
kubectl get pods
# STATUS: Pending

kubectl describe pod standard-pod
# Warning  FailedScheduling: 1 node(s) had untolerated taint {special: true}
```

## Tolerant Pod (has matching toleration)

```yaml
# tolerant-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tolerant-pod
spec:
  tolerations:
  # 1. Tolerate the custom 'special=true' taint
  - key: "special"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

  # 2. Tolerate system 'not-ready' taint for 300 seconds (default added to all pods by k8s)
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300

  containers:
  - name: nginx
    image: nginx:alpine
```

```bash
kubectl apply -f tolerant-pod.yaml
kubectl get pods -o wide
# tolerant-pod is Running on the tainted node ✅
```

## Simulate Node Crash (NoExecute)

```bash
# Manually apply the not-ready taint to simulate a crash
kubectl taint nodes <node1-name> node.kubernetes.io/not-ready:NoExecute

kubectl get pods -w
# standard-pod → terminated immediately (no toleration)
# tolerant-pod → survives for 300 seconds, then evicted
```

## Toleration Operators

| Operator | Behaviour |
|---|---|
| `Equal` | Key AND value must match exactly |
| `Exists` | Only key must exist — value doesn't matter |

## Cleanup

```bash
kubectl taint nodes <node1-name> special=true:NoSchedule-
kubectl taint nodes <node1-name> node.kubernetes.io/not-ready:NoExecute-
kubectl delete pod standard-pod tolerant-pod
```

## Notes
- `tolerationSeconds` starts counting only when the taint is actually applied
- Kubernetes auto-adds `not-ready:NoExecute` and `unreachable:NoExecute` tolerations (300s) to every pod by default
- Control plane has `node-role.kubernetes.io/control-plane:NoSchedule` taint by default
