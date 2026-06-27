# Q98. Install Nginx Ingress Controller

## Task
Deploy the Nginx Ingress Controller using official manifests. Verify:
- Controller pods are running
- IngressClass is created
- Test with a sample Ingress

---

## Phase 1: Deploy Using Official Manifests

```bash
# Apply the official bare-metal manifest (used in CKA lab environments)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/baremetal/deploy.yaml
```

> For cloud environments replace `baremetal` with `cloud`:
> ```bash
> kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
> ```

---

## Phase 2: Verify Controller Pods Are Running

```bash
kubectl get pods -n ingress-nginx
```

Expected output:
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-xxxxx        0/1     Completed   0          60s
ingress-nginx-admission-patch-xxxxx         0/1     Completed   0          60s
ingress-nginx-controller-6b94c75899-abcde   1/1     Running     0          60s
```

Wait until the controller is `Running`:
```bash
kubectl rollout status deployment ingress-nginx-controller -n ingress-nginx
```

---

## Phase 3: Verify IngressClass Was Created

```bash
kubectl get ingressclass
```

Expected output:
```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       60s
```

```bash
kubectl describe ingressclass nginx
```

> The IngressClass `nginx` is what you reference in Ingress resources with
> `spec.ingressClassName: nginx`. Without it, the controller ignores your Ingress.

---

## Phase 4: Check the Controller Service

```bash
kubectl get service -n ingress-nginx
```

```
NAME                                 TYPE        CLUSTER-IP    PORT(S)
ingress-nginx-controller             NodePort    10.96.x.x     80:31080/TCP,443:31443/TCP
ingress-nginx-controller-admission   ClusterIP   10.96.x.x     443/TCP
```

> On bare-metal the controller service is `NodePort`. Note the NodePorts —
> you'll use `<NODE-IP>:<NodePort>` to test Ingress rules.

---

## Phase 5: Test With a Sample Ingress

### Deploy a test backend

```bash
kubectl create deployment hello --image=nginx
kubectl expose deployment hello --port=80
```

### Create a sample Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: hello.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello
            port:
              number: 80
EOF
```

### Verify Ingress

```bash
kubectl get ingress hello-ingress
```

```
NAME            CLASS   HOSTS               ADDRESS         PORTS   AGE
hello-ingress   nginx   hello.example.com   192.168.1.10    80      30s
```

> `ADDRESS` shows the node IP assigned by the controller. If empty, wait 30–60s.

### Test

```bash
# Get the NodePort for HTTP
HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')

NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')

# Use Host header to simulate DNS
curl -H "Host: hello.example.com" http://$NODE_IP:$HTTP_PORT
```

Expected:
```html
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
```

---

## Summary: Key Commands

```bash
# Install
kubectl apply -f https://...ingress-nginx.../baremetal/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get ingressclass
kubectl get svc -n ingress-nginx

# Check controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

---

## Cleanup

```bash
kubectl delete ingress hello-ingress
kubectl delete deployment hello
kubectl delete service hello
```
