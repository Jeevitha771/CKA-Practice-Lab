# Q58. Create VerticalPodAutoscaler (VPA) for a Deployment

## Task
Create a VerticalPodAutoscaler for a deployment. Configure update mode and resource policies.

## Prerequisites — Install VPA

```bash
# Check if VPA CRD exists
kubectl get crd | grep verticalpodautoscaler

# Install VPA (if not present)
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify VPA components
kubectl get pods -n kube-system | grep vpa
# Expected:
#   vpa-admission-controller
#   vpa-recommender
#   vpa-updater
```

## Setup Target Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx
```

```bash
kubectl apply -f deploy.yaml
```

## VPA YAML

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  # Target — which workload to manage
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp

  # Update Policy — how/when to apply recommendations
  updatePolicy:
    updateMode: Auto   # Off | Initial | Auto | Recreate

  # Resource Guardrails
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 500m
        memory: 512Mi
      controlledResources:
      - cpu
      - memory
      controlledValues: RequestsOnly
```

```bash
kubectl apply -f vpa.yaml
kubectl get vpa
kubectl describe vpa webapp-vpa
```

## VPA Update Modes

| Mode | Behaviour |
|---|---|
| `Off` | Advisor only — calculates but never applies |
| `Initial` | Applies only when new pods are created; never restarts running pods |
| `Auto` | Actively evicts pods to apply updated resource sizes |
| `Recreate` | Explicitly recreates pods (similar to Auto in modern clusters) |

## VPA Field Reference

| Field | Purpose |
|---|---|
| `targetRef` | Points to the workload (Deployment / StatefulSet / DaemonSet) |
| `updateMode` | Controls how VPA applies recommendations |
| `minAllowed` | Floor — VPA never recommends below this |
| `maxAllowed` | Ceiling — VPA never recommends above this |
| `controlledResources` | Which resources VPA can change (`cpu`, `memory`) |
| `controlledValues` | `RequestsOnly` or `RequestsAndLimits` |

## Notes
- VPA and HPA on CPU should **not** be used together (conflict)
- VPA + HPA on custom metrics is fine
- `kubectl top pods` must work before VPA can make recommendations
