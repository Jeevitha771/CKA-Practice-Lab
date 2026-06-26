# Q56. Configure Autoscaling Based on Custom Metrics (Requests Per Second)

## Task
Configure autoscaling based on custom metrics (requests per second).
Assume metrics-server and custom metrics API are available.

## Architecture

```
Prometheus → Prometheus Adapter → Custom Metrics API → HPA
```

## HPA with Custom Metric (requests per second)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second   # Custom metric exposed via adapter
      target:
        type: AverageValue
        averageValue: "100"              # Scale when avg RPS per pod > 100
```

## Verify Custom Metrics Are Available

```bash
# Check if custom metrics API is registered
kubectl get apiservices | grep custom.metrics

# List available custom metrics
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | python3 -m json.tool

# Check HPA status
kubectl get hpa web-app-hpa
kubectl describe hpa web-app-hpa
```

## Notes
- Standard `metrics-server` only provides CPU and Memory
- Custom metrics require a **Prometheus Adapter** or similar component
- `type: Pods` → metric is averaged across all pods of the deployment
- `type: Object` → metric comes from a specific Kubernetes object (e.g. Ingress)
- `type: External` → metric from outside the cluster (e.g. queue length from SQS)
