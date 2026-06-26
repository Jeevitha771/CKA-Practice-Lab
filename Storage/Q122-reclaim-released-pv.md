# Q122. PV Stuck in Released State — Reclaim and Make Available for New Claims

## Task
PV is stuck in `Released` state. Reclaim it and make it available for new claims.

## Background — How PV Gets Stuck in Released

```bash
# 1. PVC was deleted
kubectl delete pvc myclaim

# 2. PV status changes from Bound → Released
kubectl get pv
# STATUS = Released (claimRef still points to deleted PVC)
```

## Fix — Clear the claimRef

### Fastest Way (Exam)

```bash
kubectl patch pv pv-data -p '{"spec":{"claimRef": null}}'

# Verify
kubectl get pv
# STATUS should now be: Available
```

### Alternative — kubectl edit

```bash
kubectl edit pv pv-data
# Delete the entire claimRef section:
#   claimRef:
#     namespace: default
#     name: myclaim
```

## Full Reproduce + Fix Demo

```bash
# Create PV
kubectl apply -f pv.yaml       # STATUS: Available

# Create PVC
kubectl apply -f pvc.yaml      # STATUS: Bound

# Delete PVC
kubectl delete pvc myclaim     # PV STATUS: Released

# Fix
kubectl patch pv pv-data -p '{"spec":{"claimRef": null}}'

# Verify
kubectl get pv
# NAME      CAPACITY   STATUS      CLAIM
# pv-data   5Gi        Available
```

## Notes
- `Retain` reclaim policy → PV goes to `Released` after PVC deletion (data preserved, manual cleanup needed)
- `Delete` reclaim policy → PV and underlying storage are deleted automatically
- A `Released` PV cannot be re-bound until claimRef is removed
