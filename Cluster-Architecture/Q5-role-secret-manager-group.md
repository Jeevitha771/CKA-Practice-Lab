# Q5. Role `secret-manager` Bound to a Group

## Task
In namespace `finance`:
1. Create Role `secret-manager` allowing create, update, delete on secrets
2. Bind it to group `finance-admins`

---

## Solution

This question introduces another excellent real-world scenario: managing sensitive resources (Secrets) and assigning permissions to a **Group** rather than a single user or robot.

Here is how to explain this concept to your students, followed by the commands to solve it.

## Part 1: The Theory (What's New)

Let's adapt our "office building" analogy for these two new concepts.

### 1. The Resource (`secrets`)

* **What it is:** A Kubernetes Secret is an object that contains a small amount of sensitive data such as passwords, tokens, or SSH keys.
* **The Analogy:** If Pods are filing cabinets, Secrets are the **company safes**. They require a much higher level of security clearance. Notice that the prompt asks for `create, update, delete` but *not* `get` or `list`. This is a common security pattern: an automated pipeline might be allowed to write or rotate passwords in the safe, but not read them!

### 2. The Subject (`group`)

* **What it is:** Instead of a `User` (one human) or a `ServiceAccount` (one application), a `Group` represents a collection of users defined by your identity provider (like Active Directory, Okta, or AWS IAM).
* **The Analogy:** Instead of handing the job description to "John," we are handing it to a team alias, like "The Accounting Directors." Anyone who belongs to that team automatically inherits the permissions.

---

## Part 2: Step-by-Step Solution

We can accomplish all of this quickly using imperative commands.

### Step 1: Create the Namespace

Ensure the `finance` department exists.

```bash
kubectl create namespace finance

```

### Step 2: Create the Role

Create the job description in the `finance` namespace that allows managing the safes.

```bash
kubectl create role secret-manager \
  --verb=create,update,delete \
  --resource=secrets \
  -n finance

```

### Step 3: Create the RoleBinding

Now, bind the role to the group. We don't have a specific name for the binding in the prompt, so we will call it `finance-admins-binding`.

```bash
kubectl create rolebinding finance-admins-binding \
  --role=secret-manager \
  --group=finance-admins \
  -n finance

```

*(Remind the students: Notice we use `--group=` instead of `--user=` or `--serviceaccount=`. Kubernetes handles the rest!)*

---

## Part 3: How to Verify the Solution

Testing group permissions with `kubectl auth can-i` requires a new trick. You must simulate a user, but also simulate that the user belongs to a specific group using the `--as-group` flag.

**Test 1: Can a user in the group create a secret?**

```bash
kubectl auth can-i create secrets \
  --as=dummy-user \
  --as-group=finance-admins \
  -n finance

```

*Expected Output: `yes*`

**Test 2: Can a user in the group READ (get) a secret?**
*(Since we only granted create, update, and delete, this should fail!)*

```bash
kubectl auth can-i get secrets \
  --as=dummy-user \
  --as-group=finance-admins \
  -n finance

```

*Expected Output: `no*`

**Test 3: Does it fail if they aren't in the group?**

```bash
kubectl auth can-i create secrets \
  --as=dummy-user \
  -n finance

```

*Expected Output: `no*`

## Equivalent YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-manager
  namespace: finance
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-manager-binding
  namespace: finance
subjects:
- kind: Group                   # binding to a group — not a user or SA
  name: finance-admins
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: secret-manager
  apiGroup: rbac.authorization.k8s.io
```

## Subject Types in RBAC

| Kind | Example | Notes |
|---|---|---|
| `User` | `john@example.com` | Individual user (from cert CN or OIDC) |
| `Group` | `finance-admins` | Group of users (from cert O or OIDC) |
| `ServiceAccount` | `app-sa` | Pod identity; requires `namespace` field |
