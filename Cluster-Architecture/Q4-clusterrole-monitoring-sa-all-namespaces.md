# Q4. Grant ServiceAccount Cluster-Wide Pod Access Across All Namespaces

## Task
Grant ServiceAccount `monitoring-sa` in namespace `monitoring` cluster-wide permissions to list and get pods in ALL namespaces using ClusterRole and ClusterRoleBinding.

---

## Solution

This question highlights one of the most important design features in Kubernetes RBAC: mixing cluster-scoped roles with namespace-scoped accounts. This is a classic exam question because it tests whether you know how to give a local application global reach.

Here is how to explain this cross-boundary concept to your students, followed by the exact commands.

## Part 1: The Theory (What's New)

Let's return to our "office building" analogy to explain how an identity in one department can have building-wide access.

### 1. The Cross-Boundary Concept

* **What it is:** A ServiceAccount lives in a specific namespace, but a `ClusterRoleBinding` can grant that account permissions across *every* namespace.
* **The Analogy:** The `monitoring-sa` robot officially "lives" in the Monitoring Department (the `monitoring` namespace). That is where its charging station and ID badge are registered. However, the robot's job is to audit the filing cabinets (`pods`) across the *entire* building.
* **The Rule:** To give a local robot building-wide access, you must give it a building-wide job description (`ClusterRole`) and attach it using a building-wide assignment (`ClusterRoleBinding`).

### 2. The Exam Trap (RoleBinding vs. ClusterRoleBinding)

* *Crucial detail for your students:* If you accidentally use a regular `RoleBinding` to attach a `ClusterRole` to an account, Kubernetes will silently restrict those permissions to a single namespace. To get access to *all* namespaces, the binding itself **must** be a `ClusterRoleBinding`.

---

## Part 2: Step-by-Step Solution

Because we are creating cluster-wide permissions, the Role and the Binding will not use the `-n` flag, but the ServiceAccount still needs it.

### Step 1: Create the Namespace and ServiceAccount

The robot needs its home department and its ID badge.

```bash
kubectl create namespace monitoring
kubectl create serviceaccount monitoring-sa -n monitoring

```

### Step 2: Create the ClusterRole

We need a building-wide job description. The question doesn't specify a name, so we will call it `global-pod-reader`.

```bash
kubectl create clusterrole global-pod-reader \
  --verb=get,list \
  --resource=pods

```

### Step 3: Create the ClusterRoleBinding

Now we attach the building-wide role to our namespaced robot. We will name the binding `monitoring-sa-global-binding`.

```bash
kubectl create clusterrolebinding monitoring-sa-global-binding \
  --clusterrole=global-pod-reader \
  --serviceaccount=monitoring:monitoring-sa

```

*(Remind the students: Even though the binding is cluster-wide, the ServiceAccount still lives in a specific namespace. You MUST specify it as `namespace:name` — in this case, `monitoring:monitoring-sa`).*

---

## Part 3: How to Verify the Solution

To prove this worked, your students need to test the permissions against a namespace that the ServiceAccount does *not* live in (like `default` or `kube-system`).

**Test 1: Can it list pods in the default namespace?**

```bash
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitoring-sa -n default

```

*Expected Output: `yes*`

**Test 2: Can it list pods in the kube-system namespace?**

```bash
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:monitoring-sa -n kube-system

```

*Expected Output: `yes*`

**Test 3: Can it list pods cluster-wide (all namespaces)?**

```bash
kubectl auth can-i list pods -A --as=system:serviceaccount:monitoring:monitoring-sa

```

*Expected Output: `yes*`

**Test 4: Negative Test (Can it delete a pod?)**

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:monitoring-sa -n default

```

*Expected Output: `no*`
