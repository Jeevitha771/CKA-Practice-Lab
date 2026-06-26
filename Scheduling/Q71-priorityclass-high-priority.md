# Q71. Create PriorityClass `high-priority` and Use It in a Pod

## Task
Create PriorityClass `high-priority`: value=1000, globalDefault=false.
Create pod using this PriorityClass.

## Solution

### Step 1: Create the PriorityClass

```yaml
# high-priority-class.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Use for critical service pods only."
```

```bash
kubectl apply -f high-priority-class.yaml
kubectl get priorityclass
```

### Step 2: Create Pod Using the PriorityClass

```yaml
# high-priority-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-high-priority
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
```

```bash
kubectl apply -f high-priority-pod.yaml
kubectl get pod nginx-high-priority -o yaml | grep priorityClassName
kubectl describe pod nginx-high-priority | grep Priority
```

## How PriorityClass Affects Scheduling

### Phase 1: Queue Ordering
When multiple Pods are pending, the scheduler sorts them by priority.
Pod with `value: 1000` moves to the **front of the queue** ahead of Pods with `value: 0`.

### Phase 2: Preemption (when cluster is full)
If no node has enough resources for the high-priority pod:
1. Scheduler finds nodes with **lower-priority running pods**
2. **Evicts** those lower-priority pods
3. Places the high-priority pod in the freed space

## Key Fields

| Field | Purpose |
|---|---|
| `value` | Higher = more important. Standard pods default to 0 |
| `globalDefault: false` | Must be explicitly requested in pod spec |
| `globalDefault: true` | Applied to ALL pods that don't specify a PriorityClass |
| `priorityClassName` | Pod field linking it to the PriorityClass |

## Reference
https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/
