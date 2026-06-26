# Q70. Calculate If Pod Can Schedule on Node

## Task
Calculate if pod (500m CPU, 1Gi memory) can schedule on node:
- Total node: 2 CPU / 4Gi memory
- Existing pods: 1.2 CPU / 2.5Gi memory
- System reserved: 100m CPU / 500Mi memory

## Step-by-Step Calculation

### Step 1: Convert All Units

| Resource | Value | Converted |
|---|---|---|
| Node total CPU | 2 CPU | **2000m** |
| Node total memory | 4Gi | **4096Mi** |
| Existing pods CPU | 1.2 CPU | **1200m** |
| Existing pods memory | 2.5Gi | **2560Mi** |
| System reserved CPU | 100m | **100m** |
| System reserved memory | 500Mi | **500Mi** |
| New pod CPU request | 500m | **500m** |
| New pod memory request | 1Gi | **1024Mi** |

### Step 2: Calculate Allocatable (Capacity − System Reserved)

```
Allocatable CPU    = 2000m - 100m  = 1900m
Allocatable Memory = 4096Mi - 500Mi = 3596Mi
```

### Step 3: Calculate Available (Allocatable − Existing Pods)

```
Available CPU    = 1900m - 1200m = 700m
Available Memory = 3596Mi - 2560Mi = 1036Mi
```

### Step 4: Compare Available vs New Pod Request

| Resource | Available | Requested | Fits? |
|---|---|---|---|
| CPU | 700m | 500m | ✅ Yes |
| Memory | 1036Mi | 1024Mi | ✅ Yes |

**Conclusion: Yes, the pod CAN be scheduled on this node.**

## The Golden Formula

```
Allocatable = Node Capacity - System Reserved
Available   = Allocatable - Existing Pod REQUESTS
Schedules   = Available >= New Pod REQUEST (both CPU and Memory)
```

## Exam Traps

| Trap | Rule |
|---|---|
| Mixed units (CPU + m, Gi + Mi) | Always convert: 1 CPU = 1000m, 1Gi = 1024Mi |
| Requests vs Limits | Scheduler uses **Requests** only — ignore Limits |
| Pod stays Pending | Available < Requested → `FailedScheduling: Insufficient cpu/memory` |

## Verify on a Real Node

```bash
kubectl describe node <node-name>
# Look for: "Allocated resources" section
# Shows: CPU Requests, Memory Requests vs Capacity

kubectl top node <node-name>
# Shows live utilization (requires metrics-server)
```
