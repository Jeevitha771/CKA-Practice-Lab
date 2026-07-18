# Q104. Ingress with Default Backend for Custom 404 Page

## Task
Create an Ingress with a default backend that serves a custom 404 page
when no routing rules match the incoming request.

> **SKIM note:** Know the `defaultBackend` field location and structure.
> This is an edge case but can appear as a quick task on the exam.

---

This is a fantastic demonstration to run for students. Teaching Ingress in a completely open environment (like a raw Minikube cluster, kind, or bare-metal setup) is challenging because **Kubernetes does not come with an Ingress Controller out of the box.** If a student tries to create an Ingress resource without a controller running, absolutely nothing will happen.

Here is the complete, end-to-end solution to build and demonstrate a custom 404 default backend from scratch.

---

### Step 1: Install the NGINX Ingress Controller

In an open environment, you must install a controller first so the cluster knows how to process Ingress resources.

Run this command to install the official NGINX Ingress Controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml

```

*Wait a few moments and verify the controller pod is running:*

```bash
kubectl get pods -n ingress-nginx

```

### Step 2: Create the "Custom 404" Application

For a live demonstration, it is best to use a lightweight image that outputs a clear, custom text message so students immediately know it worked. We will use `hashicorp/http-echo` for this.

Run these imperative commands to create the 404 Pod and expose it via a Service:

```bash
# 1. Create the backend pod that outputs our custom error message
kubectl run custom-404 --image=hashicorp/http-echo --port=5678 -- -text="CUSTOM 404 ERROR: The page you are looking for does not exist in this cluster."

# 2. Expose it internally on port 80
kubectl expose pod custom-404 --name=custom-404-svc --port=80 --target-port=5678

```

### Step 3: Create a "Main" Application

To prove the Ingress works properly, we need a normal application to route valid traffic to. We will use standard NGINX.

```bash
# 1. Create the main app pod
kubectl run main-app --image=nginx --port=80

# 2. Expose it
kubectl expose pod main-app --name=main-app-svc --port=80

```

### Step 4: Create the Ingress Resource

Now we tie it all together. The secret to this requirement is the `defaultBackend` field. By placing it at the root of the `spec` (outside of the `rules` array), you are telling the Ingress controller: *"If the user asks for a path that is not defined in my rules, send them here."*

Create a file named `ingress-404.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx
  # ----------------------------------------------------
  # THE SOLUTION: This catches all unmatched traffic
  # ----------------------------------------------------
  defaultBackend:
    service:
      name: custom-404-svc
      port:
        number: 80
  # ----------------------------------------------------
  rules:
  - host: demo.local
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: main-app-svc
            port:
              number: 80

```

Apply the configuration:

```bash
kubectl apply -f ingress-404.yaml

```
The Fix: You need to tell the Ingress Controller to rewrite the path to / before sending it to the pod. Run this command to add the rewrite annotation:
Bash

> kubectl annotate ingress demo-ingress nginx.ingress.kubernetes.io/rewrite-target="/"

Now, if you run curl -H "Host: demo.local" http://$INGRESS_IP/app, the Ingress will strip /app, send / to the pod, and you will see the "Welcome to nginx!" HTML.

### Step 5: Test and Demonstrate

```markdown
# Testing Ingress Default Backend (Custom 404)

Since you are testing inside the Killercoda shell, the easiest way to test routing without modifying DNS or `/etc/hosts` is to send requests directly to the Ingress Controller's internal IP address while manually passing the `Host` header.

## 1. Get the Ingress Controller IP
First, save the Ingress Controller's ClusterIP to an environment variable so the testing commands are clean and easy to read.

```bash
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.clusterIP}')

```

## 2. Test the Valid Route (Main App)

Send a request to the exact host (`demo.local`) and path (`/app`) defined in your Ingress rules.

```bash
curl -H "Host: demo.local" http://$INGRESS_IP/app

```

**Expected Output:**
You should see the raw HTML of the default "Welcome to nginx!" page. This proves your primary routing rule is working perfectly.

```

```
