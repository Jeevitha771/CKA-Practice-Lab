# Q20–Q22. etcd Cluster Health, Member Management

## Q20 Task
Check health of all etcd members: endpoint health, member list with IDs, endpoint status with leader info.

## Q21 Task
Remove a failed etcd member from a 3-node cluster. Verify remaining cluster achieves quorum.

## Q22 Task
Add a new etcd member to an existing 3-node etcd cluster. Verify the new member joins successfully.

---

## Setup: etcdctl Alias

```bash
alias etcdctl='ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'
```

---

## Q20: Check etcd Cluster Health

### Endpoint Health
```bash
etcdctl endpoint health \
  --endpoints=https://192.168.1.101:2379,https://192.168.1.102:2379,https://192.168.1.103:2379
```
```
https://192.168.1.101:2379 is healthy: committed proposal
https://192.168.1.102:2379 is healthy: committed proposal
https://192.168.1.103:2379 is healthy: committed proposal
```

### Member List with IDs
```bash
etcdctl member list --write-out=table \
  --endpoints=https://192.168.1.101:2379
```
```
+------------------+---------+----------------+----------------------------+----------------------------+
|        ID        | STATUS  |      NAME      |         PEER ADDRS         |        CLIENT ADDRS        |
+------------------+---------+----------------+----------------------------+----------------------------+
| 8e9e05c52164694d | started | control-plane-1| https://192.168.1.101:2380 | https://192.168.1.101:2379 |
| a9a4c8cf69f6a651 | started | control-plane-2| https://192.168.1.102:2380 | https://192.168.1.102:2379 |
| 5b8c0ef7dd0a9e12 | started | control-plane-3| https://192.168.1.103:2380 | https://192.168.1.103:2379 |
+------------------+---------+----------------+----------------------------+----------------------------+
```

### Endpoint Status (Shows Leader)
```bash
etcdctl endpoint status --write-out=table \
  --endpoints=https://192.168.1.101:2379,https://192.168.1.102:2379,https://192.168.1.103:2379
```
```
+----------------------------+------------------+-------+---------+-----------+...+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER |...|IS RAFT LEADER|
+----------------------------+------------------+-------+---------+-----------+...+--------+
| https://192.168.1.101:2379 | 8e9e05c52164694d | 3.5.9 |  3.2 MB |    true   |...|  true  |
| https://192.168.1.102:2379 | a9a4c8cf69f6a651 | 3.5.9 |  3.2 MB |   false   |...|  false |
| https://192.168.1.103:2379 | 5b8c0ef7dd0a9e12 | 3.5.9 |  3.2 MB |   false   |...|  false |
+----------------------------+------------------+-------+---------+-----------+...+--------+
```

---

## Q21: Remove a Failed etcd Member

```bash
# Step 1: Get the member ID of the failed node
etcdctl member list

# Step 2: Remove the failed member using its ID
etcdctl member remove 5b8c0ef7dd0a9e12

# Step 3: Verify cluster still has quorum (2/3 remaining)
etcdctl endpoint health \
  --endpoints=https://192.168.1.101:2379,https://192.168.1.102:2379

# Step 4: Verify member list updated
etcdctl member list
```

> ✅ A 3-node cluster can lose 1 member and still maintain quorum (2 of 3 needed).

---

## Q22: Add a New etcd Member

```bash
# Step 1: Register the new member (run on existing etcd cluster)
etcdctl member add control-plane-4 \
  --peer-urls=https://192.168.1.104:2380

# Output:
# Member added: ID = 7f9c1a2b3d4e5f60
# ETCD_NAME=control-plane-4
# ETCD_INITIAL_CLUSTER=control-plane-1=...,control-plane-2=...,control-plane-4=https://192.168.1.104:2380
# ETCD_INITIAL_CLUSTER_STATE=existing

# Step 2: Start etcd on the new node using the output values
# (Pass the ETCD_INITIAL_CLUSTER and ETCD_INITIAL_CLUSTER_STATE as flags or env vars)

# Step 3: Verify the new member joined
etcdctl member list
# new member should show as "started"
```

---

## etcd Quorum Reference

| Cluster Size | Failures Tolerated |
|---|---|
| 1 | 0 |
| 3 | 1 |
| 5 | 2 |
| 7 | 3 |
