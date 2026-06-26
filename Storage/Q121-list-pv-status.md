# Q121. List All PVs and Identify Their Status

## Task
List all PVs and identify which are: Available, Bound, Released, Failed.

## Solution

```bash
# List all PVs with status
kubectl get pv

# Filter by specific status
kubectl get pv | grep Available
kubectl get pv | grep Bound
kubectl get pv | grep Released
kubectl get pv | grep Failed
```

## PV Status Lifecycle

| Status | Meaning |
|---|---|
| `Available` | PV exists and has not been claimed by any PVC |
| `Bound` | PV is bound to a PVC |
| `Released` | PVC was deleted, but PV still holds the old claimRef (not yet available for new claims) |
| `Failed` | PV failed automatic reclamation |

## Example Output

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM             STORAGECLASS
pv-data   5Gi        RWO            Retain           Available                     manual
pv-db     10Gi       RWO            Delete           Bound       default/db-pvc    fast-storage
pv-old    5Gi        RWO            Retain           Released    default/old-pvc   manual
```

## Notes
- A PV in `Released` state cannot be claimed by a new PVC until the `claimRef` is cleared
- Use `kubectl patch pv <name> -p '{"spec":{"claimRef": null}}'` to move Released → Available
