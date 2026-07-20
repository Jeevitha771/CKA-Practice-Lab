# Q138. Debug Node Reporting NotReady

## Task
Node reporting `NotReady`. Debug: check node conditions, kubelet logs, container runtime status, system resources (CPU/memory/disk).

---
To demonstrate this effectively in a live Killercoda environment, you first need to intentionally break the cluster so your students have a realistic scenario to troubleshoot.

The most reliable way to simulate a `NotReady` state without destroying the environment is to stop the `kubelet` service on the worker node. Here is the exact sequence to run for your demonstration, including the setup and the teaching track.

1. **The Setup: Intentionally Break the Node:** Do this before or right at the start of the lesson.
In a standard Killercoda Kubernetes playground, you will have a control plane (`controlplane`) and a worker node (`node01`). SSH into the worker node and stop the kubelet daemon.

```bash
# Connect to the worker node
ssh node01

# Stop the kubelet service to simulate a failure
systemctl stop kubelet

# Exit back to the control plane terminal
exit

```

*Teaching note:* The Kubernetes controller manager takes about 40 seconds to realize the node has stopped reporting. You can use this time to introduce the topic to your students.


2. **Show the Symptoms:** The Hook.
Once the timeout period passes, show the students the current state of the cluster.

```bash
kubectl get nodes

```

Point out that `node01` has transitioned to `NotReady`. Explain that when this happens, no new pods can be scheduled there, and existing pods might be evicted if the outage lasts longer than 5 minutes.


3. **Investigate from the API Server:** Control Plane Diagnostics.
Walk your students through asking the cluster what it knows about the failure.

```bash
kubectl describe node node01

```

Tell the students to scroll to the **Conditions** section. Show them that the `Ready` condition has changed to `Unknown` and the reason states `NodeStatusUnknown`. Explain that this specific message almost always means the control plane can no longer communicate with the node's kubelet.


4. **Dive into the Worker Node:** Node-Level Diagnostics.
Explain that since the API server can't reach the node, you must connect to the node directly to check system resources and services.

```bash
ssh node01

```

Run through the resource checks quickly to rule them out:

```bash
# Check if disk space is 100% full (simulate DiskPressure)
df -h

# Check if memory is exhausted (simulate MemoryPressure)
free -m

```

Since resources are healthy in this clean environment, guide them to the next logical culprit: the services.


5. **Check Container Runtime and Kubelet:** The Root Cause.
First, verify the container runtime is active.

```bash
systemctl status containerd

```

Show them it is running perfectly. Next, check the kubelet.

```bash
systemctl status kubelet

```

Point out the `Active: inactive (dead)` status. To show them how a real engineer verifies *why* it stopped, check the logs.

```bash
journalctl -u kubelet -n 20 --no-pager

```


6. **Fix the Issue and Verify:** The Resolution.
Restart the service to resolve the simulated failure.

```bash
systemctl start kubelet

# Verify it is running again
systemctl status kubelet

# Exit back to the control plane
exit

```

Finally, prove to the students that the fix worked.

```bash
kubectl get nodes -w

```

The `-w` (watch) flag is great for demonstrations, as the students will see the exact moment `node01` flips from `NotReady` back to `Ready`.

kubectl top node node01
kubectl top pods -A

top -b -n 1 | head -n 5

---

## Debug Checklist

| Check | Command | Healthy Sign |
|---|---|---|
| Node conditions | `kubectl describe node <name>` | All False except Ready=True |
| kubelet service | `systemctl status kubelet` | `active (running)` |
| kubelet logs | `journalctl -u kubelet -n 50` | No ERROR lines |
| containerd | `systemctl status containerd` | `active (running)` |
| Disk space | `df -h` | < 85% used |
| Memory | `free -h` | Available > 0 |
| CPU | `top -bn1` | Not 100% sustained |
| Swap | `swapon --show` | Empty (swap disabled) |
