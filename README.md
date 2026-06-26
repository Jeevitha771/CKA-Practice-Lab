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
├── Deployments/           # Q1–Q3, Q41–Q45
├── ConfigMaps-Secrets/    # Q46–Q52
├── Scaling/               # Q53–Q58
├── Probes-DaemonSets/     # Q59–Q66
├── Jobs-CronJobs/         # Q62–Q63
├── Resource-Management/   # Q67–Q70, Q74
├── Scheduling/            # Q71–Q73
├── Manifest-Management/   # Q75–Q79
├── Networking/            # Q80–Q84
└── Storage/               # Q119–Q136
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
