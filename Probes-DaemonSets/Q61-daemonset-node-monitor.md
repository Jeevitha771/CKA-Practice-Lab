# Q61. Create DaemonSet `node-monitor` — All Nodes Including Control Plane

## Task
Create DaemonSet `node-monitor`:
- Runs on **all nodes including control plane**
- Image: `prometheus/node-exporter`
- Mounts host `/proc`, `/sys`, `/root`
- Uses `hostNetwork` and `hostPID`

## Concept

```
Control Plane  ---> 1 Pod (node-monitor)
Worker-1       ---> 1 Pod (node-monitor)
Worker-2       ---> 1 Pod (node-monitor)
Worker-3       ---> 1 Pod (node-monitor)
```

- **DaemonSet** = 1 Pod per Node automatically (vs Deployment = N pods anywhere)
- **hostNetwork** = Pod uses Node's IP (not its own virtual IP)
- **hostPID** = Pod can see all processes on the host
- **hostPath** = Pod can read host filesystem (e.g. `/proc`, `/sys`)
- **Toleration** = Bypasses the control plane's `NoSchedule` taint

## Solution YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      # Uses hostNetwork and hostPID (placed under pod spec, NOT container spec)
      hostNetwork: true
      hostPID: true

      # Toleration to run on control plane (bypasses NoSchedule taint)
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule

      containers:
      - name: node-exporter
        image: prometheus/node-exporter
        volumeMounts:
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: root
          mountPath: /host/root

      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /root
```

```bash
kubectl apply -f daemonset.yaml

# Verify one pod per node
kubectl get pods -l app=node-monitor -o wide

# Should show pods on ALL nodes including controlplane
```

## Exam Checklist

- ✅ `kind: DaemonSet`
- ✅ `tolerations` with `node-role.kubernetes.io/control-plane: NoSchedule`
- ✅ `hostNetwork: true` and `hostPID: true` under pod `spec` (not inside container)
- ✅ `volumes` (hostPath) + `volumeMounts` mapped correctly
