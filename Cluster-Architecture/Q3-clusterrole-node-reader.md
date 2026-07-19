# Q3. ClusterRole `node-reader` and ClusterRoleBinding for User

## Task
1. Create ClusterRole `node-reader` that can get, list, watch nodes cluster-wide
2. Create ClusterRoleBinding `node-reader-binding` for user `john@example.com`

---

## Solution

This question introduces the difference between standard, namespace-scoped resources and **cluster-scoped** resources.

Here is how to explain the leap from `Role` to `ClusterRole` to your students, followed by the exact commands to solve it.

## Part 1: The Theory (What's New)

We can build directly on the "office building" analogy from the previous exercises.

### 1. ClusterRole vs. Role

* **What it is:** A `Role` only works inside a specific namespace. A `ClusterRole` works across the entire cluster.
* **The Analogy:** A `Role` is a job description restricted to a single floor (the `production` department). A `ClusterRole` is a job description for the entire building—like a "Building Inspector" or "Head of Security."

### 2. Nodes (A Cluster-Scoped Resource)

* **What it is:** Nodes are the physical or virtual machines running your cluster. Because they hold the entire cluster together, they do not belong to any single namespace.
* **The Analogy:** You can't put the building's physical foundation inside the Accounting department. The foundation holds up the whole building. Therefore, to give someone permission to look at the foundation (`nodes`), you *must* use a building-wide job description (`ClusterRole`).

### 3. User vs. ServiceAccount

* **What it is:** Instead of a `ServiceAccount` (which is for applications/machines), we are binding to a `User` (a human).
* **The Analogy:** John is a human employee, not a delivery robot. When we bind the permissions, we don't need to specify what department he belongs to, because human identities in Kubernetes are global.

---

## Part 2: Step-by-Step Solution

Because this is cluster-wide, notice that we **drop the `-n` (namespace) flag completely** from our commands.

### Step 1: Create the ClusterRole

We create the building-wide job description for reading nodes.

```bash
kubectl create clusterrole node-reader --verb=get,list,watch --resource=nodes

```

### Step 2: Create the ClusterRoleBinding

We attach the building-wide job description to the human user, `john@example.com`.

```bash
kubectl create clusterrolebinding node-reader-binding \
  --clusterrole=node-reader \
  --user=john@example.com

```

*(Notice we use `--user=` instead of `--serviceaccount=`. We also don't need the `namespace:name` prefix, just the email/username).*

---

## Part 3: How to Verify the Solution

Your students can verify this using the exact same `auth can-i` command, but with two small adjustments:

1. Drop the namespace flag.
2. The `--as` flag is much simpler for humans—just provide the username directly.

**Test 1: Can John list nodes?**

```bash
kubectl auth can-i list nodes --as=john@example.com

```

*Expected Output: `yes*`

**Test 2: Negative Test (Can John delete nodes?)**

```bash
kubectl auth can-i delete nodes --as=john@example.com

```

*Expected Output: `no*`

**Test 3: Scope Test (Can John read pods?)**
Even though John has cluster-wide access, he only has access to *nodes*.

```bash
kubectl auth can-i get pods --as=john@example.com -n default

```

*Expected Output: `no*`

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: User
  name: john@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

## Role vs ClusterRole

| Feature | Role | ClusterRole |
|---|---|---|
| Scope | Single namespace | Cluster-wide |
| Can grant access to nodes | ❌ (nodes are cluster-scoped) | ✅ |
| Can grant access to namespaced resources | ✅ | ✅ (all namespaces) |
| Used with RoleBinding | ✅ | ✅ (limits to one namespace) |
| Used with ClusterRoleBinding | ❌ | ✅ (cluster-wide) |
