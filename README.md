# CKA Practice Lab

A structured collection of CKA (Certified Kubernetes Administrator) practice questions with solutions, organized by topic.

## 🚀 Start Here

| File | Description |
|---|---|
| 📋 **[CKA-Master-Commands-Cheatsheet.md](CKA-Master-Commands-Cheatsheet.md)** | **All commands in one file** — 18 sections, exam-ready reference |
| 📄 **[CKA_Exam_Practice_Questions.pdf](CKA_Exam_Practice_Questions.pdf)** | 190 exam-simulated questions (Kubernetes v1.28+) |

## 📁 Folder Structure

```
CKA-Practice-Lab/
├── Cluster-Architecture/  # Q1–Q37   ← NEW
├── Deployments/           # Q38–Q45
├── ConfigMaps-Secrets/    # Q46–Q52
├── Scaling/               # Q53–Q58
├── Probes-DaemonSets/     # Q59–Q66
├── Jobs-CronJobs/         # Q62–Q63
├── Resource-Management/   # Q67–Q70, Q74
├── Scheduling/            # Q71–Q73
├── Manifest-Management/   # Q75–Q79
├── Networking/            # Q80–Q118
├── Storage/               # Q119–Q137
└── Troubleshooting/       # Q138–Q190 ← NEW
```

---

## 📋 Question Index

### Deployments
| File | Topic |
|---|---|
| Q1 | Create Deployment `web-app` with resources and rolling update strategy |
| Q2 | Update deployment image, monitor rollout, rollback if issues |
| Q3 | Rollout history, inspect revision, rollback to specific revision |
| Q41 | Implement Canary Deployment (v1 stable + v2 canary) |
| Q42 | minReadySeconds, progressDeadlineSeconds, rolling update strategy |
| Q43 | Pause rollout, batch multiple changes, resume as single rollout |
| Q44 | Scale deployment using kubectl scale and kubectl edit |
| Q45 | Revision history limit — retain last 5 ReplicaSets |

### ConfigMaps & Secrets
| File | Topic |
|---|---|
| Q46 | Create ConfigMap from literals, file, and env-file |
| Q47 | Create Secret and mount as read-only volume |
| Q48 | Pod using ConfigMap key as env var, Secret as env vars, ConfigMap as volume |
| Q49 | Update ConfigMap and reload pods without manual deletion |
| Q50 | Create TLS Secret from certificate files |
| Q51 | Debug pod failing due to missing ConfigMap |
| Q52 | Create immutable ConfigMap and Secret |

### Scaling & Autoscaling
| File | Topic |
|---|---|
| Q53 | Scale deployment from 3 to 8 replicas, verify endpoints |
| Q54 | HorizontalPodAutoscaler — CPU 60%, Memory 80%, min 2 / max 10 |
| Q55 | Scale StatefulSet, verify sequential scaling and unique PVCs |
| Q56 | HPA based on custom metrics (requests per second) |
| Q57 | Debug HPA not scaling — metrics server, resource requests |
| Q58 | VerticalPodAutoscaler — update modes and resource policies |

### Probes, DaemonSets & Init Containers
| File | Topic |
|---|---|
| Q59 | Deployment with liveness, readiness, and startup probes |
| Q60 | Init container waiting for database service on port 5432 |
| Q61 | DaemonSet on all nodes including control plane (hostNetwork, hostPID) |
| Q64 | Debug pod restarting due to failed liveness probes — fix config |
| Q65 | StatefulSet with Parallel pod management and RollingUpdate partition |
| Q66 | Pod with multiple sequential init containers |

### Jobs & CronJobs
| File | Topic |
|---|---|
| Q62 | Job: 10 completions, 3 parallel, backoffLimit 4, deadline 600s |
| Q63 | CronJob: daily 2AM backup, Forbid concurrency, history limits |

### Resource Management
| File | Topic |
|---|---|
| Q67 | Namespace `limited` with ResourceQuota (CPU, memory, pods, services, PVCs) |
| Q68 | LimitRange in namespace `dev` with defaults, min, max |
| Q69 | Debug pod pending "Insufficient cpu" — investigate and resolve |
| Q70 | Calculate if pod can schedule on node (capacity math) |
| Q74 | Pod with requests and limits for CPU, memory, and ephemeral storage |

### Scheduling
| File | Topic |
|---|---|
| Q71 | PriorityClass `high-priority` value=1000, pod using it |
| Q72 | Pod affinity: prefer SSD nodes, require same zone, anti-affinity |
| Q73 | Taints and Tolerations: NoSchedule, NoExecute, tolerationSeconds |

### Manifest Management
| File | Topic |
|---|---|
| Q75 | Convert imperative command to YAML |
| Q76 | Generate YAML (dry-run) for Deployment, Service, ConfigMap |
| Q77 | kubectl dry-run=client vs dry-run=server — validation difference |
| Q78 | kubectl diff to preview changes before applying |
| Q79 | Kustomize: manage dev/staging/prod configs with base + overlays |

### Networking
| File | Topic |
|---|---|
| Q80 | Document cluster network config: CNI plugin, Pod CIDR, Service CIDR, DNS IP |
| Q81 | Debug CNI — pod cannot reach other pods: check plugin, routes, connectivity |
| Q82 | Pod with hostNetwork=true — verify node IP, security implications, use cases |
| Q83 | Verify IP forwarding and br_netfilter module — fix if missing |
| Q84 | Configure pod for specific network interface on multi-homed node |
| 📖 CONCEPTS | Advanced Networking deep dive: multi-homed nodes, overlay vs direct routing, Services internals (DNAT, conntrack, iptables/IPVS/eBPF), DNS flow end-to-end |
| 📖 CONCEPTS | Linux Networking Fundamentals: pause container, veth pairs, bridges, Pod CIDR, CNI internals, kube-proxy, conntrack, CoreDNS DNS flow, flat network & NetworkPolicy |

### Storage (PV / PVC / Volumes)
| File | Topic |
|---|---|
| Q119 | Create PersistentVolume with hostPath, manual StorageClass, Retain policy |
| Q120 | Create StorageClass with AWS EBS provisioner, WaitForFirstConsumer |
| Q121 | List all PVs and identify their status |
| Q122 | Reclaim PV stuck in Released state |
| Q123 | PV with ReadWriteMany — access mode comparison table |
| Q124 | PV with Block volumeMode — volumeDevices in pod |
| Q125 | Change PV reclaim policy from Delete to Retain |
| Q126 | PVC with selector matching PV label |
| Q127 | Pod using PVC — write file, delete pod, verify data persists |
| Q128 | Debug PVC stuck in Pending state |
| Q129 | Expand PVC from 5Gi to 10Gi |
| Q130 | PVC binding to specific PV — selector vs volumeName |
| Q131 | StatefulSet with VolumeClaimTemplates (10Gi per pod) |
| Q132 | Deployment with 4 volume types: ConfigMap, Secret, PVC, EmptyDir |
| Q133 | Init container writing to shared emptyDir volume |
| Q134 | Backup Job copying data from source PVC to backup PVC |
| Q135 | hostPath volume for logging — security implications |
| Q136 | Projected volume combining ConfigMap, Secret, DownwardAPI |
| Q137 | StatefulSet with volumeClaimTemplates — unique PVC per pod verified |

### Cluster Architecture, Installation & Configuration *(NEW)*
| File | Topic |
|---|---|
| Q1 | Create ServiceAccount, Role `pod-reader`, RoleBinding in `production` |
| Q2 | Role with `pods/log` and `pods/exec` subresources — `dev-user` |
| Q3 | ClusterRole `node-reader` + ClusterRoleBinding for user |
| Q4 | ServiceAccount cluster-wide pod read access across all namespaces |
| Q5 | Role `secret-manager` bound to group `finance-admins` |
| Q6 | Check and grant deployment permissions to ServiceAccount |
| Q7 | ClusterRole `deployment-manager` — full permissions on deployments + replicasets |
| Q8 | ServiceAccount read-only access to ConfigMaps and Secrets cluster-wide |
| Q9 | Role for pods, services, endpoints — bound to ServiceAccount `app-viewer` |
| Q10 | Enforce create-but-not-delete pod permission for a user |
| Q11 | Initialize Kubernetes cluster with kubeadm (pod CIDR, version, endpoint) |
| Q12 | Generate kubeadm join command for worker node |
| Q13 | Install Calico CNI and verify pod-to-pod communication |
| Q14–Q15 | Bootstrap tokens — create, list, delete expired |
| Q16 | kubeadm configuration file (custom CIDR, endpoint, version) |
| Q17 | HA cluster with 3 control planes + HAProxy load balancer |
| Q18 | Backup etcd with etcdctl snapshot save |
| Q19 | Restore etcd from backup — update manifest, verify cluster |
| Q20–Q22 | etcd member management: health check, remove failed, add new |
| Q23 | Firewall ports for control plane and worker nodes |
| Q24 | Verify containerd — service status, version, crictl, image pull |
| Q25 | Configure kubelet (clusterDNS, clusterDomain, maxPods) |
| Q26–Q27 | Verify prerequisites (swap, kernel modules, sysctl) + cgroup driver |
| Q28–Q30 | Cluster upgrade v1.27 → v1.28: control plane + workers |
| Q31–Q32 | Rollback failed worker upgrade; upgrade control plane only |
| Q33–Q35 | Automated etcd backup script, DR test, verify backup without restoring |
| Q36–Q37 | Change etcd data directory; remote backup with scp |

### Troubleshooting *(NEW)*
| File | Topic |
|---|---|
| Q138 | Debug Node `NotReady` — conditions, kubelet logs, runtime, disk |
| Q139–Q140 | Pod logging: all patterns, previous crash, label selector, multi-container |
| Q141 | Debug `CrashLoopBackOff` — logs, describe, exit codes, common causes |
| Q142 | Kubelet not starting — certs, cgroup driver, config, swap |
| Q143–Q145 | `kubectl top` — nodes, pods, containers; install metrics-server |
| Q146–Q147 | OOMKilled — analyze and fix; deploy metrics-server for all nodes |
| Q148–Q151 | Log streaming, multi-container logs, dual logging pod, missing logs debug |
| Q152 | Systematically debug application not responding (7-layer checklist) |
| Q153–Q158 | Stuck rollout, 503 errors, local-vs-k8s failure, pending pod, DB connectivity, intermittent |
| Q159–Q166 | API server, scheduler, controller-manager, kubelet, CoreDNS, kube-proxy, etcd, certs |
| Q167–Q174 | Cross-node networking, service LB, ingress 404, DNS failure, NodePort, NetworkPolicy, intermittent |
| Q175–Q183 | Full stack debug, post-upgrade, high load, monitoring setup, secure namespace, migrate stateful app, cert expiry, Istio, DR drill |
| Q184–Q190 | Cluster optimization, cloud PV provisioning, complex networking, custom scheduler, health check, eviction, Pod Security Standards |
