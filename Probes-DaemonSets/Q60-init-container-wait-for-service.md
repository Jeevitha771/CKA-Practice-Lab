# Q60. Init Container: Wait for Service `database` on Port 5432 Before Starting Main App

## Task
Create pod with init container that:
- Waits for service `database` to be accessible on port 5432
- Times out after 60 seconds
- Then starts the main container

## Full Solution

```yaml
# PIECE 1: The actual Postgres Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "mysecretpassword"

---
# PIECE 2: The Database Service (creates DNS entry "database")
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ClusterIP
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  selector:
    app: postgres

---
# PIECE 3: App Pod with Init Container
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
  labels:
    app: my-app
spec:
  initContainers:
  - name: wait-for-database
    image: busybox:1.36
    command:
      - "/bin/sh"
      - "-c"
      - "timeout 60 sh -c 'until nc -z database 5432; do echo Waiting for database...; sleep 2; done; echo Database is up!'"

  containers:
  - name: main-app
    image: nginx:alpine
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f postgres-deploy.yaml
kubectl apply -f database-service.yaml
kubectl apply -f app-pod.yaml

# Watch init container complete, then main container start
kubectl get pod app-with-init -w

# Check init container logs
kubectl logs app-with-init -c wait-for-database
```

## Notes
- `nc -z <host> <port>` — checks if a TCP port is open without sending data
- `timeout 60` — aborts if the loop doesn't complete within 60 seconds
- The init container loops every 2 seconds until the database service responds
- Main container only starts after all init containers exit with code 0

## Reference
https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
