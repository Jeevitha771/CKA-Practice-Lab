# Q136. Projected Volume — Combine ConfigMap, Secret, and DownwardAPI in One Mount

## Task
Create a pod with a projected volume that combines ConfigMap, Secret, and DownwardAPI under a single mount path.

## Setup

```bash
# Create ConfigMap
kubectl create configmap myconfigmap --from-literal=config="env=prod"
kubectl get configmap

# Create Secret
kubectl create secret generic mysecret \
  --from-literal=username=admin \
  --from-literal=password=welcome123
kubectl get secret
```

## Pod YAML with Projected Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-pod
spec:
  containers:
  - name: app
    image: busybox:1.36
    command:
    - sh
    - -c
    - sleep 3600
    volumeMounts:
    - name: projected-volume
      mountPath: /projected-data

  volumes:
  - name: projected-volume
    projected:
      sources:

      # Source 1: ConfigMap
      - configMap:
          name: myconfigmap

      # Source 2: Secret
      - secret:
          name: mysecret

      # Source 3: DownwardAPI — pod metadata as files
      - downwardAPI:
          items:
          - path: pod-name
            fieldRef:
              fieldPath: metadata.name

          - path: pod-namespace
            fieldRef:
              fieldPath: metadata.namespace
```

```bash
kubectl apply -f pod.yaml

# List all projected files
kubectl exec projected-pod -- ls /projected-data

# Read ConfigMap value
kubectl exec projected-pod -- cat /projected-data/config

# Read Secret value
kubectl exec projected-pod -- cat /projected-data/password

# Read pod name from DownwardAPI
kubectl exec projected-pod -- cat /projected-data/pod-name

# Read namespace from DownwardAPI
kubectl exec projected-pod -- cat /projected-data/pod-namespace
```

## What a Projected Volume Does

Instead of creating 3 separate volume mounts, a `projected` volume merges all sources into a **single directory**:

```
/projected-data/
├── config          ← from ConfigMap (myconfigmap)
├── username        ← from Secret (mysecret)
├── password        ← from Secret (mysecret)
├── pod-name        ← from DownwardAPI (metadata.name)
└── pod-namespace   ← from DownwardAPI (metadata.namespace)
```

## Supported Projected Sources

| Source | What It Provides |
|---|---|
| `configMap` | ConfigMap keys as files |
| `secret` | Secret keys as files |
| `downwardAPI` | Pod/container metadata (name, namespace, labels, resource limits) |
| `serviceAccountToken` | Automatically projected SA token |
