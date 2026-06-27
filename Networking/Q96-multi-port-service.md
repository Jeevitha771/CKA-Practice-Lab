# Q96. Create Multi-Port Service: HTTP, HTTPS, Metrics

## Task
Create a service that exposes multiple ports:
- Port `80` → targetPort `8080` (HTTP)
- Port `443` → targetPort `8443` (HTTPS)
- Port `9090` → targetPort `9090` (metrics)

---

## Phase 1: Deploy a Multi-Port Application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-port-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multi-port-app
  template:
    metadata:
      labels:
        app: multi-port-app
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8443
          name: https
        - containerPort: 9090
          name: metrics
EOF
```

---

## Phase 2: Create the Multi-Port Service

### Important Rule
When a service exposes **multiple ports**, every `ports` entry **must** have a `name` field.
Single-port services don't require a name — multi-port services do.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: multi-port-svc
spec:
  type: ClusterIP
  selector:
    app: multi-port-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
EOF
```

**Field breakdown:**

| Name | Port | TargetPort | Purpose |
|---|---|---|---|
| `http` | `80` | `8080` | Web traffic |
| `https` | `443` | `8443` | TLS web traffic |
| `metrics` | `9090` | `9090` | Prometheus scraping |

---

## Phase 3: Verify the Service

```bash
kubectl get service multi-port-svc
```

```
NAME              TYPE        CLUSTER-IP    PORT(S)                      AGE
multi-port-svc    ClusterIP   10.96.x.x     80/TCP,443/TCP,9090/TCP      20s
```

```bash
kubectl describe service multi-port-svc
```

```
Name:              multi-port-svc
Selector:          app=multi-port-app
Type:              ClusterIP
IP:                10.96.x.x
Port:              http    80/TCP
TargetPort:        8080/TCP
Port:              https   443/TCP
TargetPort:        8443/TCP
Port:              metrics 9090/TCP
TargetPort:        9090/TCP
Endpoints:         10.244.x.x:8080,10.244.x.x:8080
                   10.244.x.x:8443,10.244.x.x:8443
                   10.244.x.x:9090,10.244.x.x:9090
```

---

## Phase 4: Verify Endpoints

```bash
kubectl get endpoints multi-port-svc
```

---

## Phase 5: Test Each Port

```bash
kubectl run test --image=busybox:1.28 --rm -it -- sh
```

Inside the shell:

```sh
# Test HTTP port
wget -qO- multi-port-svc:80

# Test metrics port
wget -qO- multi-port-svc:9090

# Test by port name using DNS (works with named ports)
wget -qO- multi-port-svc:80
```

---

## Using Named Ports in Other Resources

Named ports are useful in NetworkPolicies and Ingress rules:

```yaml
# NetworkPolicy referencing port by name
spec:
  ingress:
  - ports:
    - port: http     # uses the named port from the service
```

```yaml
# Readiness probe referencing named container port
readinessProbe:
  httpGet:
    port: http       # refers to containerPort name, not number
```

---

## Summary: Quick Reference

```bash
# Create
kubectl apply -f multi-port-svc.yaml

# Verify
kubectl get svc multi-port-svc
kubectl describe svc multi-port-svc
kubectl get endpoints multi-port-svc

# Test
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- multi-port-svc:80
kubectl run test --image=busybox:1.28 --rm -it -- wget -qO- multi-port-svc:9090
```

---

## Cleanup

```bash
kubectl delete service multi-port-svc
kubectl delete deployment multi-port-app
```
