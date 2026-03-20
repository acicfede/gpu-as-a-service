# GPU-as-a-Service: Kueue-based GPU Quota Management

Multi-tenant GPU quota management for OpenShift AI using Kueue, deployed via ArgoCD with Kustomize.

## Architecture Overview

```
                         ┌──────────────┐
                         │  gpu-cohort   │
                         │   (Cohort)    │
                         │  4 GPUs total │
                         └──────┬───────┘
                                │
                ┌───────────────┼───────────────┐
                │                               │
       ┌────────┴─────────┐           ┌─────────┴────────┐
       │   inference-cq    │           │   training-cq     │
       │  (ClusterQueue)   │           │  (ClusterQueue)   │
       │                   │           │                   │
       │  Guaranteed: 3 GPU│           │  Guaranteed: 1 GPU│
       │  Borrow:  up to 1 │           │  Borrow:  up to 2 │
       │  Max:     4 GPUs  │           │  Max:     3 GPUs  │
       │  Priority: 1000   │           │  Priority: 100    │
       └────────┬──────────┘           └─────────┬─────────┘
                │                                │
       ┌────────┴──────────┐           ┌─────────┴─────────┐
       │   team-inference   │           │   team-training    │
       │   (LocalQueue)     │           │   (LocalQueue)     │
       │   ns: team-inference│          │   ns: team-training│
       └────────┬──────────┘           └─────────┬─────────┘
                │                                │
       ┌────────┴──────────┐           ┌─────────┴─────────┐
       │   highpriority     │           │   lowpriority      │
       │ (HardwareProfile)  │           │ (HardwareProfile)  │
       │   GPU: 1-4         │           │   GPU: 1-3         │
       └───────────────────┘           └────────────────────┘
```

## Cluster Capacity

| Resource | Total |
|----------|-------|
| Physical GPUs (NVIDIA L4) | 4 |
| GPU Nodes | 1 |

## GPU Quota Allocation

| ClusterQueue | Guaranteed GPUs | Borrowing Limit | Max Usable | Priority |
|---|---|---|---|---|
| `inference-cq` | 3 | 1 | 4 | 1000 (high) |
| `training-cq` | 1 | 2 | 3 | 100 (low) |
| **Cohort total** | **4** | | | |

## Scheduling Scenarios

### Only inference active
Inference uses up to **4 GPUs** (3 guaranteed + 1 borrowed from training's idle quota).

### Only training active
Training uses up to **3 GPUs** (1 guaranteed + 2 borrowed from inference's idle quota).

### Both teams active
- Inference gets **3 GPUs** (its guaranteed quota).
- Training gets **1 GPU** (its guaranteed quota).
- Total: 4, matching physical capacity.

## Preemption Rules

| Rule | inference-cq | training-cq |
|------|---|---|
| `reclaimWithinCohort` | `Any` | `LowerPriority` |
| `borrowWithinCohort` | `LowerPriority` (threshold: 100) | Not set |
| `withinClusterQueue` | `LowerPriority` | `LowerPriority` |

### What this means

- **Inference can preempt training to borrow GPUs.** The `borrowWithinCohort: LowerPriority` policy with `maxPriorityThreshold: 100` allows inference (priority 1000) to evict training workloads (priority 100) when it needs to borrow from the cohort.
- **Inference can always reclaim its lent quota.** With `reclaimWithinCohort: Any`, if training borrowed inference's idle GPUs and inference needs them back, inference can preempt any training workload regardless of priority.
- **Training cannot preempt inference.** Training has no `borrowWithinCohort` policy, so it can only borrow GPUs that are genuinely idle in the cohort.
- **Training reclaim is limited.** With `reclaimWithinCohort: LowerPriority`, training can only reclaim its lent quota by preempting lower-priority workloads. Since inference has higher priority (1000 > 100), training cannot reclaim from inference.

## Resource Flavors

| Flavor | Description | Node Selector | Tolerations |
|--------|-------------|---------------|-------------|
| `gpu-flavor` | NVIDIA GPU nodes | `nvidia.com/gpu.present: "true"` | `nvidia.com/gpu` (NoSchedule) |
| `default-flavor` | CPU/Memory (any node) | None | None |

## Hardware Profiles (OpenShift AI Dashboard)

| Profile | Display Name | Queue | Priority Class | GPU Range |
|---------|-------------|-------|----------------|-----------|
| `highpriority` | highPriority | `team-inference` | `inference-priority` | 1-4 |
| `lowpriority` | lowPriority | `team-training` | `low-priority` | 1-3 |

Hardware profiles integrate with the OpenShift AI dashboard, allowing users to select a profile when launching a workbench or job. The profile automatically routes the workload to the correct LocalQueue with the appropriate priority class.

## Priority Classes

| Name | Value | Description |
|------|-------|-------------|
| `inference-priority` | 1000 | High priority for inference workloads |
| `training-priority` | 100 | Low priority for training workloads |

## Directory Structure

```
base/
├── kustomization.yaml                  # Root kustomization
├── namespaces/                         # Namespace definitions
│   ├── kustomization.yaml
│   ├── team-inference-namespace.yaml
│   └── team-training-namespace.yaml
├── resourceflavor/                     # Kueue ResourceFlavors
│   ├── kustomization.yaml
│   ├── default-resourceflavor.yaml
│   └── ResourceFlavor-4GPUs.yaml
└── kueue/
    ├── queue/                          # Cohort, ClusterQueues, LocalQueues
    │   ├── kustomization.yaml
    │   ├── cohort.yaml
    │   ├── clusterqueue-inference.yaml
    │   ├── clusterqueue-training.yaml
    │   ├── localqueue-inference.yaml
    │   └── localqueue-training.yaml
    ├── priorityclass/                  # Workload priority classes
    │   ├── kustomization.yaml
    │   ├── inferencePriorityClass.yaml
    │   └── trainingPriorityClass.yaml
    └── hardware-profiles/              # OpenShift AI hardware profiles
        ├── kustomization.yaml
        ├── highPriority-hardwareProfile.yaml
        └── lowPriority-hardwareProfile.yaml
```

## Prerequisites

- OpenShift 4.x with OpenShift AI installed
- Kueue operator enabled in the DataScienceCluster
- At least one GPU node with NVIDIA GPUs
- ArgoCD for GitOps deployment

## Deployment

### Via ArgoCD

Create an ArgoCD Application pointing to this repository with path `base`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gpu-as-a-service
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
  source:
    repoURL: <your-repo-url>
    path: base
    targetRevision: main
  project: default
```

### Manual

```bash
oc apply -k base/
```
