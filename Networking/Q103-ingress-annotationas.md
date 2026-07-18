# Q103. Configure Ingress Annotations ΓÇö SSL Redirect, Timeout, Body Size, Rate Limiting

## Task
Configure an Ingress with Nginx annotations for:
- SSL redirect (HTTP ΓåÆ HTTPS)
- Connection timeout: `60s`
- Proxy body size: `10m`
- Rate limiting: `10 req/s`

> **SKIM note:** Annotation specifics are Nginx-controller-dependent. For the CKA exam,
> know the annotation format and where to add them. Deep memorisation is not required.

---
It is completely normal for us to panic about memorizing these specific annotations. In a stressful exam environment, pulling exact string keys from memory is risky.

Since the official Kubernetes documentation (`kubernetes.io/docs`) only provides basic Ingress examples and doesn't heavily list NGINX-specific annotations, we need a reliable strategy.

Here is how you teach them to "remember" the annotations using logic, followed by the complete open-environment demonstration.

---

### The "Cheat Code" for Remembering Ingress Annotations

 not to memorize the whole string, but to understand the **Anatomy of an Annotation**. Every NGINX Ingress annotation follows a strict formula:
`[The Universal Prefix] + [The Logical Action]`

**1. The Universal Prefix**
Every single annotation for this controller starts with the exact same prefix. If they can remember this, they are 80% of the way there:
`nginx.ingress.kubernetes.io/`

**2. The Logical Action (The Suffix)**
Instead of memorizing, teach them to ask: *"What is NGINX actually doing here?"*

* **SSL:** We want to *force* traffic to redirect to SSL. $\rightarrow$ `force-ssl-redirect: "true"`
* **Timeouts & Sizes:** NGINX is acting as a *proxy*. Therefore, these settings always start with `proxy-`.
* Read timeout? $\rightarrow$ `proxy-read-timeout: "120"`
* Body size? $\rightarrow$ `proxy-body-size: "50m"`


* **Rate Limiting:** We want to *limit* the *Requests Per Second (RPS)*. $\rightarrow$ `limit-rps: "10"`

---

### The Open-Environment Demonstration

Assuming you have already installed the NGINX Ingress Controller (as done in the previous step), here is the end-to-end demonstration to prove these annotations actually control traffic.

#### Step 1: Deploy a Target Application

First, spin up a basic backend application so the Ingress has somewhere to send the traffic.

```bash
# Create the pod
kubectl run secure-app --image=nginx --port=80

# Expose it with a service
kubectl expose pod secure-app --name=secure-app-svc --port=80

```

#### Step 2: Apply the Ingress Manifest

Create a file named `annotated-ingress.yaml`. This file combines the Universal Prefix with the Logical Actions we mapped out above.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # 1. SSL Redirect: Forces HTTP traffic to redirect to HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # 2. Connection/proxy timeouts (in seconds)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
    # 3. Max allowed request body size (e.g. for file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    # Rate limiting: 10 requests per second per IP
    # Timeout: Sets the time (in seconds) to wait for a backend response
    nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: secure.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app-svc
            port:
              number: 80

```

Apply the file to the cluster:

```bash
kubectl apply -f annotated-ingress.yaml

```

#### Tests

Find your Ingress Controller's IP address (for this example, we will assume it is exposed locally at `127.0.0.1`). Run these specific commands to prove that the annotations are enforcing the rules.

**Test 1: Prove SSL Redirect is Active**
We will send a basic HTTP request and look at the headers to ensure the controller rejects it and demands HTTPS.

```bash
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.clusterIP}')
curl -I -H "Host: secure.local" http://$INGRESS_IP/

```

*What we will see:* An `HTTP/1.1 308 Permanent Redirect` header, proving `force-ssl-redirect` is working.

**Test 2: Prove Body Size Limits**
By default, the NGINX Ingress Controller applies a burst multiplier of 5 to your rate limit. This is designed to handle sudden legitimate traffic spikes without instantly dropping connections.

Because you set limit-rps: "10", NGINX calculates a burst queue of 50 (10 x 5). Since your bash loop only sent 20 requests, they were all absorbed by the burst queue and processed successfully.
Fire 100 rapid requests over HTTPS.
for i in {1..100}; do curl -k -s -o /dev/null -w "%{http_code}\n" -H "Host: secure.local" https://$INGRESS_IP/; done
You should see a cluster of 200 responses followed immediately by 503 (Service Temporarily Unavailable) once the 10-request limit is breached in that second.

---

## 1. SSL Redirect

    The Annotation: nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    What it does: It intercepts any traffic coming in on the insecure HTTP port (Port 80) and immediately sends a 308 Permanent Redirect response to the client, telling their browser to try again using HTTPS (Port 443).

    Why it's necessary: In modern web architecture, you never want users submitting plaintext data. Even if a user manually types http:// into their browser, this annotation acts as a bouncer, ensuring they cannot interact with your backend pods until their connection is encrypted.

## 2. Connection Timeout (Proxy Read Timeout)

    The Annotation: nginx.ingress.kubernetes.io/proxy-read-timeout: "60" (Note: Your task specifies 60s, while the yaml example used 120s!)

    What it does: Once NGINX passes a user's request to your backend pod, it starts a timer. This annotation dictates exactly how many seconds NGINX will wait for the pod to send back a response.

    Why it's necessary: Without this, if your backend database hangs or crashes mid-request, NGINX would keep the connection open indefinitely. If this happens thousands of times, NGINX runs out of worker connections and the whole Ingress controller crashes. This annotation ensures NGINX gracefully cuts the cord and returns a 504 Gateway Timeout to the user, freeing up resources.

## 3. Proxy Body Size

    The Annotation: nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    What it does: It looks at the Content-Length header of an incoming HTTP request (usually a POST or PUT request where a user is uploading data) and rejects the request if the payload is larger than the specified size (10 Megabytes in your task).

    Why it's necessary: This is a crucial security and stability measure. If your application only processes small text forms, a malicious user or bot could try to upload a 50GB file to exhaust your server's memory or storage (a type of Denial of Service attack). This tells NGINX to block large payloads at the door with a 413 Request Entity Too Large error before it ever reaches your application.

## 4. Rate Limiting (RPS)

    The Annotation: nginx.ingress.kubernetes.io/limit-rps: "10"

    What it does: NGINX tracks the IP address of every incoming request. If a single IP address makes more than 10 requests in a single second, NGINX temporarily blocks them.

    Why it's necessary: This prevents "noisy neighbors" and brute-force attacks. If a bot starts hammering your login page with 500 password attempts per second, this annotation ensures they are immediately throttled and handed a 503 Service Unavailable error, keeping the application fast and responsive for normal, legitimate users.
   
