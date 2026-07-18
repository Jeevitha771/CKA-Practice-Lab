# Q98. Install Nginx Ingress Controller

## Task
Deploy the Nginx Ingress Controller using official manifests. Verify:
- Controller pods are running
- IngressClass is created
- Test with a sample Ingress

---
## Solution:
This is a very realistic and common scenario. If you are running Kubernetes on your own hardware (bare-metal), a local virtual machine, or a tool like `kubeadm` or `Minikube`, you do not have AWS or Google Cloud sitting in the background to hand you a public IP address.

If you use the standard cloud installation manifest in this environment, the Service will ask for a `LoadBalancer` IP, and it will sit in a `<pending>` state forever because nobody is there to answer the request.

Here is exactly how to solve this using the official **Bare-Metal** approach, which relies on a `NodePort` instead of a `LoadBalancer`.

---

### **Step 1: Install Using the Bare-Metal Manifest**

The NGINX team provides a specific manifest for non-cloud environments. Instead of requesting a Load Balancer, it exposes the Ingress Controller on your actual worker nodes using a **NodePort** (a port in the 30000ΓÇô32767 range).

Run this command:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

```

---

### **Step 2: Verify Pods and IngressClass**

You still need to ensure the receptionist is hired and sitting at the desk.

**1. Check the Pods:**
Ensure the controller pod is `Running` and ready (`1/1`).

```bash
kubectl get pods -n ingress-nginx

```

**2. Check the IngressClass:**
Verify the `nginx` class was created so you can assign rules to it.

```bash
kubectl get ingressclass

```

---

### **Step 3: Deploy the Sample App and Ingress Rule**

The application and the Ingress rule are **exactly the same** as they would be in the cloud. The beauty of the Ingress Controller is that your internal apps don't care where the external traffic comes from.

**1. Create the App and Service:**

```bash
kubectl create deployment demo-app --image=nginxdemos/hello:plain-text
kubectl expose deployment demo-app --name=demo-service --port=80

```

**2. Create the Ingress Rule (Using the fast imperative method):**

```bash
kubectl create ingress demo-ingress --rule="demo.local/=demo-service:80" --class=nginx

```

---

### **Step 4: The Crucial Difference ΓÇö How to Test It**

Because you do not have a single Cloud External IP, you must access the Ingress Controller by hitting the IP address of **any of your Kubernetes worker nodes** on the specific **NodePort** that was assigned during installation.

**1. Find the NodePort:**
Run this command to look at the Ingress Controller's service:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller

```

Look at the `PORT(S)` column. It will look something like this:
`80:31234/TCP, 443:32567/TCP`

The first number (`80`) is the internal port. The second number (`31234`) is your **NodePort**. Write that number down.

**2. Find your Node's IP address:**
If you don't know the IP of your machine/node, you can find it by running:

```bash
kubectl get nodes -o wide

```

Look under the `INTERNAL-IP` column. Let's assume the IP is `192.168.1.50`.

**3. Test the Routing:**
Now, you combine them. You send the `curl` request to your Node's IP and the NodePort, while still passing the Host header so the Ingress Controller knows which rule to match.

```bash
curl -H "Host: demo.local" http://192.168.1.50:31234

```

If it works, you will immediately see the plain-text response from the `demo-app` container. You have successfully routed external traffic into a bare-metal cluster without spending a dime on a cloud provider!

---

### Or:

## Phase 1: Deploy Using Official Manifests

```bash
# Apply the official bare-metal manifest (used in CKA lab environments)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

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

> On bare-metal the controller service is `NodePort`. Note the NodePorts ΓÇö
> you'll use `<NODE-IP>:<NodePort>` to test Ingress rules.

---

## Phase 5: Test With a Sample Ingress

### Deploy a test backend

```bash
kubectl create deployment hello --image=nginx
kubectl expose deployment hello --port=80
or
kubectl expose deployment hello --port=8090 --target-port=80 (in other examples)
```

or 

```bash
kubectl create deployment demo-app --image=nginxdemos/hello:plain-text
kubectl expose deployment demo-app --name=demo-service --port=80
```

### Create a sample Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
spec:
  ingressClassName: nginx  # Choose your receptionist! (e.g., 'nginx' or 'nginx-internal')
  rules:
  - http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: hello
            port:
              number: 8090   # This MUST match the --port from your expose command
EOF
```
or

Here is the exact imperative command to create that same Ingress without writing the YAML file:

```bash
kubectl create ingress hello-ingress --class=nginx --rule="/hello=hello:8090"

```

### How the command maps to your YAML:

It's always helpful to see exactly how the command flags translate to the YAML you just learned:

* **`hello-ingress`**: Maps to `metadata.name`.
* **`--class=nginx`**: Maps to `spec.ingressClassName`.
* **`--rule="/hello=hello:8090"`**: This single flag handles the entire `rules` section.
* `/hello` maps to the `path` (and automatically sets the `pathType` to `Prefix`).
* `hello:8090` maps to the backend `service.name` and `service.port.number`.



*(Note: If you ever want to preview the YAML that an imperative command would generate without actually applying it to the cluster, you can add `--dry-run=client -o yaml` to the end of the command!)*
### Verify Ingress

```bash
kubectl get ingress hello-ingress
```

```
NAME            CLASS   HOSTS               ADDRESS         PORTS   AGE
hello-ingress   nginx   hello.example.com   192.168.1.10    80      30s
```

> `ADDRESS` shows the node IP assigned by the controller. If empty, wait 30ΓÇô60s.

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
## Explanation for testing:
This is the most satisfying partΓÇöproving that it actually works!

Since you are using a bare-metal setup (where the controller is exposed via `NodePort`), you can't just type `localhost` or rely on a cloud LoadBalancer IP. You need to hit the physical IP of your Kubernetes node on the specific port that the Ingress Controller is listening on.

Here is exactly how to test it in three steps.

### Step 1: Get your Node's IP Address

You need to know the IP address of the machine (the worker node) where your cluster is running. Run this command:

```bash
kubectl get nodes -o wide

```

Look under the **INTERNAL-IP** (or **EXTERNAL-IP** if applicable) column.

* *Note: If you are using Minikube for your local lab, you can just run `minikube ip` to get this address immediately.*

LetΓÇÖs assume your Node IP is `192.168.1.50`.

### Step 2: Confirm the NodePort (The Front Door)

From your previous message, we already know the `ingress-nginx-controller` Service opened port `31080` for regular HTTP traffic.

If you ever forget it, you can always check it again with:

```bash
kubectl get service ingress-nginx-controller -n ingress-nginx

```

### Step 3: Run the Test!

Now, put it all together. You want to hit your **Node IP**, on the **NodePort**, at the **/hello** path.

You can literally open your web browser and type this into the address bar:
`http://<YOUR_NODE_IP>:31080/hello`

Or, if you prefer the command line, use `curl`:

```bash
curl http://<YOUR_NODE_IP>:31080/hello

```

### What You Should See

Because you created the deployment using `--image=nginx`, you should instantly see the raw HTML for the default **"Welcome to nginx!"** landing page.

If you see that, congratulations! You have successfully configured a full traffic lifecycle:
**Browser $\rightarrow$ NodePort $\rightarrow$ Ingress Controller $\rightarrow$ Ingress Rule $\rightarrow$ Service $\rightarrow$ Pod.**

### Troubleshooting (Just in case)

* **If you see "404 Not Found" (from nginx):** Your traffic reached the Ingress Controller, but the rule didn't match. Double-check that your Ingress YAML is applied and the path is exactly `/hello`.
* **If you see "502 Bad Gateway":** The Ingress Controller accepted the traffic and tried to send it to your Pod, but the Pod rejected it. This usually means the `--target-port=80` step was missed on the Service.
* **If it just hangs/times out:** You might have a firewall blocking port `31080` on your machine, or you are using the wrong Node IP.

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
