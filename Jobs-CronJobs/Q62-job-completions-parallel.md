# Q62. Create Job: 10 Completions, 3 Parallel, Backoff Limit 4, Active Deadline 600s

## Task
Create Job with:
- 10 completions
- 3 parallel pods
- backoffLimit: 4
- activeDeadlineSeconds: 600
- Command: `echo "Processing $HOSTNAME" && sleep 10`

## Solution YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 10           # Total pods that must succeed
  parallelism: 3            # Run 3 pods at a time
  backoffLimit: 4           # Retry failed pods up to 4 times
  activeDeadlineSeconds: 600  # Hard timeout: kill entire job after 600s

  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["/bin/sh", "-c", "echo \"Processing $HOSTNAME\" && sleep 10"]
      restartPolicy: Never
```

## Imperative Shortcut (Exam Tip)

```bash
# Step 1: Generate base YAML
kubectl create job batch-processor --image=busybox \
  --dry-run=client -o yaml \
  -- /bin/sh -c 'echo "Processing $HOSTNAME" && sleep 10' > job.yaml

# Step 2: Edit job.yaml and add under spec:
#   completions: 10
#   parallelism: 3
#   backoffLimit: 4
#   activeDeadlineSeconds: 600

kubectl apply -f job.yaml

# Monitor
kubectl get job batch-processor -w
kubectl get pods -l job-name=batch-processor
```

## Field Reference

| Field | Purpose |
|---|---|
| `completions` | Total pods that must finish successfully |
| `parallelism` | How many pods run simultaneously |
| `backoffLimit` | Max retries before marking Job as Failed |
| `activeDeadlineSeconds` | Hard timeout for the entire Job |
| `restartPolicy: Never` | Create new pod on failure (not restart same container) |
