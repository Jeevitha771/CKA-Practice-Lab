# Q74. Create Pod with Resource Requests and Limits for CPU, Memory, and Ephemeral Storage

## Task
Create a pod with resource requests and limits for:
- CPU
- Memory
- Ephemeral storage

## Solution YAML

```yaml
# resource-demo-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo-pod
spec:
  containers:
  - name: my-app
    image: nginx:alpine
    resources:
      requests:
        cpu: "250m"               # Guaranteed minimum: 1/4 of a CPU
        memory: "64Mi"            # Guaranteed minimum: 64 MiB RAM
        ephemeral-storage: "1Gi"  # Guaranteed minimum: 1 GiB temp disk
      limits:
        cpu: "500m"               # Hard maximum: 1/2 of a CPU
        memory: "128Mi"           # Hard maximum: 128 MiB RAM
        ephemeral-storage: "2Gi"  # Hard maximum: 2 GiB temp disk
```

```bash
kubectl apply -f resource-demo-pod.yaml
kubectl get pod resource-demo-pod
kubectl describe pod resource-demo-pod | grep -A10 Requests
kubectl describe node <node-name> | grep -A10 "Allocated resources"
```

## What Happens When Limits Are Exceeded

| Resource | What Happens at Limit |
|---|---|
| **CPU** | Container is **throttled** (slowed down). Pod NOT killed. Performance degrades. |
| **Memory** | Container is **OOMKilled** (Out Of Memory). Pod is restarted. |
| **Ephemeral Storage** | Pod is **evicted** by Kubelet to protect node disk. |

## OOMKill Demo

```yaml
# memory-hog-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog-pod
spec:
  containers:
  - name: hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"   # App tries to use 250M but limit is 100M → OOMKilled
```

```bash
kubectl apply -f memory-hog-pod.yaml
kubectl get pods -w
# Running → OOMKilled → CrashLoopBackOff

kubectl describe pod memory-hog-pod
# Last State: OOMKilled
```

## Requests vs Limits Summary

| Concept | Who Uses It | Effect |
|---|---|---|
| `requests` | **Scheduler** | Node must have this much free before placing pod |
| `limits` | **Kubelet** | Node enforces this ceiling; CPU throttled, Memory killed, Storage evicted |

## Notes
- CPU is **compressible** → throttled at limit, never killed
- Memory is **incompressible** → OOMKilled at limit
- Ephemeral storage is **incompressible** → Pod evicted at limit
- Always set both `requests` AND `limits` — requests without limits = no protection for other pods
