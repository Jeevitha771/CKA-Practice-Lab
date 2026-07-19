# Q7. ClusterRole `deployment-manager` with Full Permissions

## Task
1. Create ClusterRole `deployment-manager` with full permissions on deployments and replicasets
2. Bind to user `admin@example.com`

---

## Solution

This question combines several concepts we have covered and introduces a powerful new shortcut: **Wildcards**.

## Part 1: The Theory (What's New)

We can build on our "office building" analogy, focusing on the Apps Wing and how we hand out permissions.

### 1. The "Master Key" (Wildcards)

* **What it is:** The prompt asks for "full permissions." Instead of typing out every single verb (`get, list, watch, create, update, patch, delete`), Kubernetes allows you to use the asterisk `*` wildcard.
* **The Analogy:** Instead of writing a job description that says, "You can look at, open, add files to, and destroy the filing cabinets," you simply give the employee a **Master Key** (`*`) to those specific cabinets.

### 2. Multiple Resources

* **What it is:** You can grant access to multiple resources in a single rule, as long as they belong to the same API group.
* **The Analogy:** Both Deployments and ReplicaSets live in the "Apps Wing" of the building. Because they are in the same wing, we can write a single job description for both of them at the same time.

---

## Part 2: Step-by-Step Solution

When using the `*` wildcard in a Linux terminal, you **must** wrap it in quotes (like `"*"`). If you don't, the Linux shell will think you are trying to list all the files in your current directory and the command will fail!

### Step 1: Create the ClusterRole

We create a building-wide job description, granting the master key (`*`) for both `deployments` and `replicasets`.

```bash
kubectl create clusterrole deployment-manager \
  --verb="*" \
  --resource=deployments,replicasets

```

*(Notice that we separate the resources with a comma and absolutely no spaces).*

### Step 2: Create the ClusterRoleBinding

We attach this building-wide job description to the human user, `admin@example.com`. We will name the binding `deployment-manager-binding`.

```bash
kubectl create clusterrolebinding deployment-manager-binding \
  --clusterrole=deployment-manager \
  --user=admin@example.com

```

---

## Part 3: How to Verify the Solution

Because this user has "full permissions" on these two resources across the entire cluster, we should test a variety of verbs and namespaces.

**Test 1: Can they create a deployment in the default namespace?**

```bash
kubectl auth can-i create deployments --as=admin@example.com -n default

```

*Expected Output: `yes*`

**Test 2: Can they delete a replicaset in the kube-system namespace?**

```bash
kubectl auth can-i delete replicasets --as=admin@example.com -n kube-system

```

*Expected Output: `yes*`

**Test 3: Scope Test (Can they create pods?)**
Even though they have a "Master Key," it only works on Deployments and ReplicaSets. They should be blocked from managing Pods directly.

```bash
kubectl auth can-i create pods --as=admin@example.com

```

*Expected Output: `no*`

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
- apiGroups: ["apps"]                        # deployments and replicasets are in "apps" group
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: deployment-manager-binding
subjects:
- kind: User
  name: admin@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

## Common API Groups

| Resource | apiGroup |
|---|---|
| pods, services, configmaps, secrets, serviceaccounts | `""` (core) |
| deployments, replicasets, daemonsets, statefulsets | `apps` |
| ingresses | `networking.k8s.io` |
| roles, rolebindings, clusterroles, clusterrolebindings | `rbac.authorization.k8s.io` |
| jobs, cronjobs | `batch` |
| horizontalpodautoscalers | `autoscaling` |
