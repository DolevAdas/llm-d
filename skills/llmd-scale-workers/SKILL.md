---
name: llm-d-scale-workers
description: Scale prefill or decode workers in existing llm-d deployments on Kubernetes/OpenShift. Supports both manual scaling (immediate one-time adjustments) and automatic scaling with WVA (continuous saturation-based autoscaling). Use for handling load changes, optimizing P/D ratios, or adjusting resource utilization without disrupting running deployments.
---

# llm-d Worker Scaling Skill

## Overview

Scale prefill and decode workers in existing llm-d deployments without full redeployment. Supports:
- **Manual scaling** - Immediate one-time adjustments via scripts, kubectl, Helm, or LeaderWorkerSet
- **Automatic scaling (WVA)** - Continuous saturation-based autoscaling for production workloads

Works with P/D disaggregation, standard inference, and LeaderWorkerSet deployments.

## When to Use

**Manual Scaling:**
- Quick adjustments for known workload changes
- Development/testing environments
- P/D disaggregation or Wide-EP deployments (WVA not yet supported)
- Immediate, predictable control needed

**Automatic Scaling (WVA):**
- Production with variable traffic patterns
- Hands-off optimization based on inference server saturation
- Intelligent Inference Scheduling deployments only (currently)
- Reduce operational overhead

## Prerequisites

- Existing llm-d deployment in Kubernetes/OpenShift
- kubectl or oc CLI with appropriate permissions
- Sufficient cluster resources (GPUs, RDMA, memory)

## Initial User Interaction

### Step 1: Detect Current Deployment

**First, detect the current deployment to understand the environment:**

```bash
# Run detection script
bash skills/llmd-scale-workers/scripts/detect-deployment.sh ${NAMESPACE}
```

This shows:
- Helm releases
- Deployments and LeaderWorkerSets
- Current decode/prefill workers
- Current replica counts
- Platform type (OpenShift, GKE, Kind, etc.)

### Step 2: Ask User to Choose Scaling Approach

**Ask:** "I've detected your deployment: [summary]

Choose scaling approach:
- **Manual** - Scale workers once now (immediate adjustment)
- **Automatic (WVA)** - Set up continuous autoscaling based on workload saturation"

#### If Automatic: Run WVA Setup

```bash
bash skills/llmd-scale-workers/scripts/setup-wva-autoscaling.sh
```

The script auto-detects config, confirms with user, installs WVA, and verifies. See [`guides/workload-autoscaling/README.md`](../../guides/workload-autoscaling/README.md) for manual setup.

**WVA handles scaling automatically once configured - no further steps needed.**

#### If Manual: Continue Below

### Step 3: Check Available Resources

```bash
# Check cluster resources
bash skills/llmd-scale-workers/scripts/check-resources.sh ${NAMESPACE}
```

### Step 4: Ask How Many Workers to Scale

**Ask the user how many workers of decode and prefill they want to add**

### Step 5: Validate and Provide Recommendations

- Calculate if the requested workers can fit in the cluster
- If resources are insufficient, inform the user and suggest the maximum possible workers
- If the user requests too many workers for available resources, suggest a realistic number
- If the P/D ratio seems suboptimal for their use case, recommend better ratios (see P/D Ratio Guidelines below)
- Example: "Your cluster has 16 available GPUs. With TP=4 for decode workers, you can add up to 4 decode workers. Would you like to proceed with 4 instead?"

### Step 6: Confirm Before Scaling

- Show the user what will be scaled and the expected resource usage
- Wait for explicit confirmation before executing the scaling commands

## Manual Scaling Workflow

### 1. Choose Scaling Method

**Ask:** "Which scaling method do you prefer?
1. **Script** (Recommended) - Automated validation, handles all types
2. **Helm/Helmfile** - Best for Helm-managed, maintains state
3. **kubectl** - Fastest, doesn't update Helm state
4. **LeaderWorkerSet** - For LWS deployments only"

**Priority by deployment type:**
- **Helm/Helmfile-managed:** Helm → Script → kubectl
- **LeaderWorkerSet:** LWS → Script → kubectl
- **Standard Deployment:** kubectl → Script
- **Uncertain:** Script (safest)

### 4. Execute Scaling

**Method A: Using Script (Recommended for most cases)**
```bash
# Scale decode workers (replace ${DECODE_COUNT} with user's desired count)
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t decode -r ${DECODE_COUNT}

# Scale prefill workers (replace ${PREFILL_COUNT} with user's desired count)
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t prefill -r ${PREFILL_COUNT}
```

**Advantages:**
- Validates resources before scaling
- Handles multiple deployment types
- Provides clear error messages
- Safer for production

**When to use:**
- Default choice for most scenarios
- When you need validation and safety checks
- When deployment type is uncertain

---

**Method B: Direct kubectl (Fastest)**
```bash
# Quick scale
#replace ${DECODE_COUNT} with user's desired count
kubectl scale deployment <deployment-name> --replicas=${DECODE_COUNT} -n ${NAMESPACE}
```

**Advantages:**
- Fastest execution
- Simple and direct
- No dependencies

**Disadvantages:**
- No resource validation
- Doesn't update Helm state
- Manual error handling

**When to use:**
- Non-Helm deployments
- Emergency scaling
- When speed is critical
- Development/testing environments

---

**Method C: Helm/Helmfile (Best for Helm-managed)**
```bash
# 1. Edit values file (e.g., guides/pd-disaggregation/ms-pd/values.yaml)
#    Change: decode.replicas: 1 → 3

# 2. Apply changes
cd guides/<guide-name>
helmfile apply -n ${NAMESPACE}
```

**Advantages:**
- Maintains Helm state consistency
- Declarative configuration
- Version controlled changes
- Rollback capability

**Disadvantages:**
- Slower execution
- Requires values file access
- More complex workflow

**When to use:**
- Helm/Helmfile-managed deployments
- Production environments
- When state consistency is critical
- When changes need to be version controlled

---

**Method D: LeaderWorkerSet (LWS-specific)**
```bash
#replace ${DECODE_COUNT} with user's desired count
kubectl scale leaderworkerset <lws-name> --replicas=${DECODE_COUNT} -n ${NAMESPACE}
```

**Advantages:**
- Native LWS scaling
- Fast and reliable
- Maintains LWS semantics

**Disadvantages:**
- Only works with LWS deployments
- Doesn't update Helm state if Helm-managed

**When to use:**
- LeaderWorkerSet deployments only
- When LWS-specific features are needed
- Wide-EP or LWS-based architectures

## Complete Workflow

### Manual Scaling
1. **Detect** - Run [`detect-deployment.sh`](scripts/detect-deployment.sh)
2. **Validate** - Run [`check-resources.sh`](scripts/check-resources.sh)
3. **Choose Method** - Ask user or use priority recommendations
4. **Scale** - Execute chosen method (Script/Helm/kubectl/LWS)
5. **Monitor** - Watch pods and verify InferencePool discovery
6. **Test** - Verify inference routing

### Automatic Scaling (WVA)
Run [`setup-wva-autoscaling.sh`](scripts/setup-wva-autoscaling.sh) which:
1. Auto-detects platform, deployments, accelerators, model ID, Prometheus
2. Confirms configuration with user
3. Installs WVA CRDs and configures platform-specific settings
4. Verifies installation (pods, HPA, metrics)
5. Provides monitoring commands

See [`guides/workload-autoscaling/README.md`](../../guides/workload-autoscaling/README.md) for manual setup.


## P/D Ratio Guidelines

Use these guidelines to recommend appropriate worker counts based on the user's workload:

**High Input/Low Output (10k ISL / 1k OSL)**
- Ratio: 4:1 or 8:1 (prefill:decode)
- More prefill workers needed
- Example: 8 prefill workers, 2 decode workers

**Low Input/High Output (1k ISL / 10k OSL)**
- Ratio: 1:2 or 1:4 (prefill:decode)
- More decode workers needed
- Example: 2 prefill workers, 8 decode workers

**Balanced (4k ISL / 4k OSL)**
- Ratio: 2:1 or 1:1 (prefill:decode)
- Example: 4 prefill workers, 2 decode workers OR 4 prefill workers, 4 decode workers

**When to Suggest Different Ratios:**
- If user requests 1 decode and 20 prefill for a high-output workload, suggest reversing the ratio
- If user requests equal workers but their workload is clearly skewed, recommend adjusting
- Always explain the reasoning based on their expected input/output sequence lengths

## Resource Requirements

**Decode Workers:**
- Higher tensor parallelism (TP=4, TP=8)
- Example: 1 worker = 4 GPUs + 1 RDMA

**Prefill Workers:**
- Lower tensor parallelism (TP=1, TP=2)
- Example: 1 worker = 1 GPU + 1 RDMA

## Monitoring and Verification

```bash
# Watch pod status
kubectl get pods -n ${NAMESPACE} -l llm-d.ai/role=decode -w

# Wait for readiness
kubectl wait --for=condition=ready pod \
  -l llm-d.ai/role=decode -n ${NAMESPACE} --timeout=600s

# Check InferencePool
kubectl describe inferencepool <pool-name> -n ${NAMESPACE}

# Test endpoint
GATEWAY_ADDR=$(kubectl get gateway <gateway-name> -n ${NAMESPACE} \
  -o jsonpath='{.status.addresses[0].value}')
curl http://${GATEWAY_ADDR}/v1/models
```

## Troubleshooting

**Pods Stuck in Pending**
```bash
# Check why
kubectl describe pod <pod-name> -n ${NAMESPACE}

# Check node resources
kubectl describe nodes | grep -A 10 "Allocated resources"
```

**Pods Failing to Start**
```bash
# Check logs
kubectl logs <pod-name> -n ${NAMESPACE} -c vllm

# Check RDMA
kubectl describe node <node-name> | grep rdma
```

**InferencePool Not Discovering Workers**
```bash
# Verify labels
kubectl get pods -n ${NAMESPACE} --show-labels

# Check InferencePool selector
kubectl get inferencepool <pool-name> -n ${NAMESPACE} -o yaml
```

## Safety Checklist

Before scaling:
- ✅ Verify current deployment state
- ✅ Check available cluster resources
- ✅ Calculate required resources
- ✅ Confirm namespace and deployment names
- ✅ Test with small increments first
- ✅ Monitor pod startup
- ✅ Verify InferencePool discovery

## Notes

- Scaling is non-destructive to existing workers
- New workers auto-join InferencePool
- Scale-down gracefully terminates workers
- Helm method maintains state consistency
- kubectl scaling is faster but doesn't update Helm state