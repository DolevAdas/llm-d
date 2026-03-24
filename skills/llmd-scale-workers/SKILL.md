---
name: llm-d-scale-workers
description: Execute scaling actions for llm-d prefill/decode workers on Kubernetes/OpenShift. Supports manual scaling (immediate adjustments), automatic WVA autoscaling (continuous saturation-based), and suspend/resume operations. Use for handling load changes, optimizing worker ratios, cost savings, or setting up autoscaling.
---

# llm-d Worker Scaling Skill

## 📋 Command Execution Notice

**Before executing any command, I will:**
1. **Explain what the command does** - Clear description of purpose and expected outcome
2. **Show the actual command** - The exact command to be executed
3. **Explain why it's needed** - How it fits into the workflow

> ## 🔔 ALWAYS NOTIFY BEFORE CREATING RESOURCES
>
> **RULE**: Before creating ANY resource (namespaces, files, Kubernetes objects), notify the user first.
>
> **Format**: "I am about to create `<resource-type>` named `<name>` because `<reason>`. Proceeding now."
>
> **Never silently create resources.** Check existence first, then notify before acting.

## Critical Rules

1. **Do NOT change cluster-level definitions** - All changes must be within the designated namespace. Never modify cluster-wide resources (ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes). Always scope commands with `-n ${NAMESPACE}`.

2. **Do NOT modify existing repository code** - Only create new files. Never edit pre-existing repository files. For customization, create new files and reference them.

## Overview

Scale prefill and decode workers in existing llm-d deployments without full redeployment:
- **Manual scaling** - Immediate adjustments via scripts or kubectl
- **Automatic scaling (WVA)** - Continuous saturation-based autoscaling for production
- **Suspend/Resume** - Scale to zero for cost savings, restore to previous state

Works with P/D disaggregation, standard inference, and LeaderWorkerSet deployments.

## When to Use

| Method | Use Cases |
|--------|-----------|
| **Manual Scaling** | Quick adjustments for known workload changes<br>Development/testing environments<br>P/D disaggregation or Wide-EP deployments<br>Immediate, predictable control needed |
| **Automatic Scaling (WVA)** | Production with variable traffic patterns<br>Hands-off optimization based on saturation<br>Intelligent Inference Scheduling deployments only<br>Reduce operational overhead |
| **Suspend/Resume** | Off-hours cost savings<br>Maintenance windows or planned downtime<br>Free up cluster resources without deletion |

## Prerequisites

- Existing llm-d deployment in Kubernetes/OpenShift
- kubectl or oc CLI with appropriate permissions
- Sufficient cluster resources (GPUs, RDMA, memory)

## Workflow

**CRITICAL RULES:**
1. **ALWAYS use existing scripts** from `skills/llmd-scale-workers/scripts/`
2. **NEVER create README.md files** - provide summaries in conversation only
3. **NEVER create new scripts** - use existing ones
4. **Scripts run non-interactively by default** - designed for automation (use `-i` flag for interactive mode)

### Step 1: Detect Deployment

```bash
bash skills/llmd-scale-workers/scripts/detect-deployment.sh ${NAMESPACE}
```

### Step 2: Execute Scaling Action

**WVA Autoscaling Setup:**
```bash
NAMESPACE=${NAMESPACE} bash skills/llmd-scale-workers/scripts/deploy-wva-controller.sh
```
See [`WVA_CONTROLLER_DEPLOYMENT.md`](skills/llmd-scale-workers/WVA_CONTROLLER_DEPLOYMENT.md) for detailed setup and troubleshooting.

**Manual Scaling:**
```bash
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT}
```

**Suspend/Resume Operations:**
```bash
# Suspend (saves current replica counts as annotations)
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r 0
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t prefill -r 0

# Resume (restore from annotations)
DECODE_REPLICAS=$(kubectl get deployment -n ${NAMESPACE} -l llm-d.ai/role=decode -o jsonpath='{.items[0].metadata.annotations.llm-d\.ai/previous-replicas}')
PREFILL_REPLICAS=$(kubectl get deployment -n ${NAMESPACE} -l llm-d.ai/role=prefill -o jsonpath='{.items[0].metadata.annotations.llm-d\.ai/previous-replicas}')

bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${DECODE_REPLICAS:-2}
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t prefill -r ${PREFILL_REPLICAS:-4}
```

### Output Format

**Correct:** Execute scripts, provide brief summary (3-5 sentences)

**WRONG:**
- ❌ Create README.md, monitoring-guide.md, or any documentation files
- ❌ Create new scripts (use existing ones)

## Scaling Methods

**Deployment Type Detection:** Auto-detected in Step 1

| Deployment Type | Scaling Method |
|----------------|----------------|
| **Standard Deployment** | `scale-workers.sh` script or `kubectl scale deployment` |
| **LeaderWorkerSet** | `kubectl scale leaderworkerset` |

**Execution Examples:**
```bash
# Using script (recommended) - auto-detects deployment type
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT}

# Interactive mode (optional)
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT} -i

# Direct kubectl (standard deployments)
kubectl scale deployment <name> --replicas=${COUNT} -n ${NAMESPACE}

# Direct kubectl (LeaderWorkerSet)
kubectl scale leaderworkerset <name> --replicas=${COUNT} -n ${NAMESPACE}
```

### Scaling Characteristics

**kubectl Scaling Benefits:**
- ✅ No pod restarts - existing pods keep in-memory vLLM cache intact
- ✅ Immediate effect - new pods added/removed without disruption
- ⚠️ **Helm-managed deployments:** kubectl scaling creates drift from values.yaml. Running `helmfile apply` later reverts to values.yaml replica count.

**Cache Implications:**
- **Existing pods:** Retain in-memory HBM prefix cache (no performance impact)
- **New pods:** Start with empty cache, require warmup period
- **Shared storage:** Tiered prefix cache with CephFS/Lustre persists across pods
- **Performance:** Expect temporary TTFT increase for new pods during warmup

**Best Practices:**
1. Use kubectl scaling for quick, non-disruptive adjustments
2. Document scaling changes for Helm-managed deployments to track drift
3. Use shared storage backends in production to minimize cache warmup impact
4. Prefer WVA autoscaling for production workloads with variable traffic
5. Always suspend both worker types together to avoid resource waste
6. Verify saved replica counts before suspending
7. Test resume in non-production first


## P/D Ratio Guidelines

Recommend worker counts based on workload characteristics:

| Workload Pattern | Ratio (P:D) | Example Configuration | Reasoning |
|-----------------|-------------|----------------------|-----------|
| **High Input/Low Output**<br>(10k ISL / 1k OSL) | 4:1 or 8:1 | 8 prefill : 2 decode | More prefill capacity needed |
| **Low Input/High Output**<br>(1k ISL / 10k OSL) | 1:2 or 1:4 | 2 prefill : 8 decode | More decode capacity needed |
| **Balanced**<br>(4k ISL / 4k OSL) | 2:1 or 1:1 | 4 prefill : 2 decode<br>OR 4 prefill : 4 decode | Balanced workload |

**Adjustment Recommendations:**
- If user requests mismatched ratios (e.g., 1 decode + 20 prefill for high-output), suggest reversing
- For equal workers with skewed workload, recommend adjusting based on ISL/OSL
- Always explain reasoning based on expected input/output sequence lengths

## Resource Requirements

| Worker Type | Tensor Parallelism | Example Resources |
|-------------|-------------------|-------------------|
| **Decode** | Higher (TP=4, TP=8) | 1 worker = 4 GPUs + 1 RDMA |
| **Prefill** | Lower (TP=1, TP=2) | 1 worker = 1 GPU + 1 RDMA |

## Post-Scaling Verification

After scaling, verify the workers are running:
```bash
kubectl get pods -n ${NAMESPACE} -l llm-d.ai/role=decode
kubectl wait --for=condition=ready pod -l llm-d.ai/role=decode -n ${NAMESPACE} --timeout=600s
```

If issues occur, check pod status with `kubectl describe pod <pod-name> -n ${NAMESPACE}`.