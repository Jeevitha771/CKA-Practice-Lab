# Q66. Pod with Multiple Sequential Init Containers

## Task
Configure a pod with multiple init containers that run sequentially to set up the environment.

## Solution YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-init-pod
spec:
  volumes:
  - name: shared-env-data
    emptyDir: {}

  # Init containers run ONE BY ONE in the order listed
  initContainers:

  # 1st Init Container
  - name: init-step-1
    image: busybox:1.28
    command:
    - /bin/sh
    - -c
    - |
      echo "Starting Step 1..."
      echo "Environment Setup: Step 1 Complete" > /env-dir/setup.txt
      sleep 5
    volumeMounts:
    - name: shared-env-data
      mountPath: /env-dir

  # 2nd Init Container (runs only after init-step-1 exits successfully)
  - name: init-step-2
    image: busybox:1.28
    command:
    - /bin/sh
    - -c
    - |
      echo "Starting Step 2..."
      echo "Environment Setup: Step 2 Complete" >> /env-dir/setup.txt
      sleep 5
    volumeMounts:
    - name: shared-env-data
      mountPath: /env-dir

  # Main container starts only after ALL init containers finish
  containers:
  - name: main-app
    image: busybox:1.28
    command:
    - /bin/sh
    - -c
    - |
      echo "Main App is starting. Reading setup files..."
      cat /env-dir/setup.txt
      sleep 3600
    volumeMounts:
    - name: shared-env-data
      mountPath: /env-dir
```

```bash
kubectl apply -f multi-init-pod.yaml

# Watch the progression
kubectl get pods multi-init-pod -w
```

## Pod Status Progression

```
Pending
  → Init:0/2   (init-step-1 running)
  → Init:1/2   (init-step-1 done, init-step-2 running)
  → PodInitializing  (all inits done, starting main app)
  → Running
```

## Check Logs Per Container

```bash
kubectl logs multi-init-pod -c init-step-1
kubectl logs multi-init-pod -c init-step-2
kubectl logs multi-init-pod -c main-app
```

## Notes
- Init containers share volumes with main containers (via `emptyDir` here)
- If any init container fails, the Pod restarts from the beginning (depending on `restartPolicy`)
- Each init container must exit with code 0 for the next one to start
- Reference: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
