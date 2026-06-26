# CKA Practice Lab

A structured collection of CKA (Certified Kubernetes Administrator) practice questions with solutions, organized by topic.

## 📁 Folder Structure

```
CKA-Practice-Lab/
├── Deployments/          # Q1–Q3, Q41–Q45
├── ConfigMaps-Secrets/   # Q46–Q52
├── Scaling/              # Q53–Q58
└── Storage/              # Q119–Q136
```

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

### Scaling
| File | Topic |
|---|---|
| Q53 | Scale deployment from 3 to 8 replicas, verify endpoints |
| Q54 | HorizontalPodAutoscaler — CPU 60%, Memory 80%, min 2 / max 10 |
| Q55 | Scale StatefulSet, verify sequential scaling and unique PVCs |
| Q56 | HPA based on custom metrics (requests per second) |
| Q57 | Debug HPA not scaling — metrics server, resource requests |
| Q58 | VerticalPodAutoscaler — update modes and resource policies |

### Storage
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
