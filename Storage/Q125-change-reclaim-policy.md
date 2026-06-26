# Q125. Change PV Reclaim Policy from Delete to Retain

## Task
Change a PV's reclaim policy from Delete to Retain. Explain implications.

## Solution

```bash
# Check current policy
kubectl get pv
kubectl describe pv <pv-name>

# Option 1: Edit the PV directly
kubectl edit pv <pv-name>
# Change:
#   spec:
#     persistentVolumeReclaimPolicy: Retain

# Option 2: Patch (faster in exam)
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Verify
kubectl get pv <pv-name> -o yaml | grep persistentVolumeReclaimPolicy
```

## Reclaim Policy Comparison

| Policy | What Happens When PVC Is Deleted | Data Safe? |
|---|---|---|
| `Delete` | PV and underlying storage deleted automatically | ❌ Data lost |
| `Retain` | PV moves to Released state, data preserved, manual cleanup needed | ✅ Data safe |
| `Recycle` | Data wiped (`rm -rf`), PV made Available again (deprecated) | ❌ Data wiped |

## Implications of Changing Delete → Retain

✅ **Benefits**
- Data on the volume is preserved after PVC deletion
- Allows manual inspection and backup before cleanup
- Protects against accidental PVC deletion

⚠️ **Drawbacks**
- PV stays in `Released` state — must be manually reclaimed
- Requires admin to clear `claimRef` before PV can be reused
- Storage is not automatically freed → potential cost/capacity issues

## Quick Reference

```
"Keep data safe"           → Retain
"Auto cleanup needed"      → Delete
Retain = manual work needed
Delete = automatic cleanup
```
