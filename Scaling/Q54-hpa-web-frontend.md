# Q54. Configure HorizontalPodAutoscaler for `web-frontend`

## Task
Configure HPA for `web-frontend`: min 2, max 10 replicas, target CPU 60%, memory 80%.
Generate load and verify autoscaling.

## Prerequisites

```bash
# Verify metrics-server is running
kubectl get deployment metrics-server -n kube-system

# If not installed
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Fix TLS issue in kubeadm clusters
kubectl edit deployment metrics-server -n kube-system
# Add under containers.args:
#   - --kubelet-insecure-tls

# Verify metrics are working
kubectl top nodes
kubectl top pods
```

## Setup Deployment with Resource Requests

```bash
kubectl create deployment web-frontend --image=nginx --replicas=2

kubectl expose deployment web-frontend --port=80 --target-port=80

kubectl edit deployment web-frontend
# Add resources block:
```

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

```bash
kubectl rollout status deployment/web-frontend
```

## HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

## Generate Load and Verify

```bash
# Create load generator
kubectl run load-generator --rm -it --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://web-frontend; done"

# Watch HPA scale up
kubectl get hpa -w
kubectl get pods -w
kubectl top pods

# Detailed HPA status
kubectl describe hpa web-frontend-hpa
```

## Service YAML (if needed)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
spec:
  selector:
    app: web-frontend
  ports:
    - port: 80
      targetPort: 80
```
