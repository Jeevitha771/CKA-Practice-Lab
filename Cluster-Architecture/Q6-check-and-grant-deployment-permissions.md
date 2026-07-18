# Q6. Check and Grant Deployment Permissions to ServiceAccount

## Task
Check if ServiceAccount `app-sa` in namespace `production` can create deployments. If not, grant the necessary permissions.

---

## Phase 1: Check Current Permissions

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:production:app-sa \
  -n production
# Expected output: no
```

---

## Phase 2: Grant the Missing Permission

```bash
# Create a Role allowing deployment creation
kubectl create role deployment-creator \
  --verb=create,get,list,watch,update,patch \
  --resource=deployments \
  -n production

# Bind the Role to the ServiceAccount
kubectl create rolebinding deployment-creator-binding \
  --role=deployment-creator \
  --serviceaccount=production:app-sa \
  -n production
```

---

## Phase 3: Verify

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:production:app-sa \
  -n production
# yes

kubectl auth can-i delete deployments \
  --as=system:serviceaccount:production:app-sa \
  -n production
# no
```

---

## Full Permission Audit for a ServiceAccount

```bash
# Check multiple verbs at once using a loop
for verb in get list watch create update patch delete; do
  echo -n "$verb deployments: "
  kubectl auth can-i $verb deployments \
    --as=system:serviceaccount:production:app-sa \
    -n production
done
```

Expected output after granting:
```
get deployments: yes
list deployments: yes
watch deployments: yes
create deployments: yes
update deployments: yes
patch deployments: yes
delete deployments: no
```

---

## Notes

- `deployments` belong to API group `apps` — `kubectl create role` handles this automatically
- To also manage ReplicaSets (which Deployments create), add `--resource=replicasets` to the same role
- Use `kubectl auth can-i --list` to see all permissions for a subject:

```bash
kubectl auth can-i --list \
  --as=system:serviceaccount:production:app-sa \
  -n production
```
