---
name: llm-d-scale-workers
description: Scale prefill or decode workers in existing llm-d deployments on Kubernetes/OpenShift without redeployment. Use when you need to add/remove workers to handle load changes, optimize P/D ratios, or adjust resource utilization dynamically without disrupting running deployments.
---

# llm-d Worker Scaling Skill

## Overview

Scale prefill and decode workers in existing llm-d deployments without full redeployment. Supports P/D disaggregation, standard inference, and LeaderWorkerSet deployments.

## When to Use

- Add/remove decode workers for throughput changes
- Add/remove prefill workers to balance P/D ratios
- Scale workers without disrupting existing deployments
- Optimize resource utilization dynamically

## Prerequisites

- Existing llm-d deployment in Kubernetes/OpenShift
- kubectl or oc CLI with appropriate permissions
- Sufficient cluster resources (GPUs, RDMA, memory)

## Initial User Interaction

1. **Ask the user how many workers of decode and prefill they want to add:**

2. **Validate the request against available resources:**
   - Run [`check-resources.sh`](scripts/check-resources.sh) to see available GPUs, RDMA, and memory
   - Calculate if the requested workers can fit in the cluster
   - If resources are insufficient, inform the user and suggest the maximum possible workers

3. **Provide recommendations if the request seems unreasonable:**
   - If the user requests too many workers for available resources, suggest a realistic number
   - If the P/D ratio seems suboptimal for their use case, recommend better ratios (see P/D Ratio Guidelines below)
   - Example: "Your cluster has 16 available GPUs. With TP=4 for decode workers, you can add up to 4 decode workers. Would you like to proceed with 4 instead?"

4. **Confirm before scaling:**
   - Show the user what will be scaled and the expected resource usage
   - Wait for explicit confirmation before executing the scaling commands

## Workflow

### 1. Detect Current Deployment

```bash
# Run detection script
bash .claude/skills/llmd-scale-workers/scripts/detect-deployment.sh ${NAMESPACE}
```

This shows:
- Helm releases
- Deployments and LeaderWorkerSets
- Current decode/prefill workers
- Current replica counts

### 2. Check Available Resources

```bash
# Check cluster resources
bash .claude/skills/llmd-scale-workers/scripts/check-resources.sh ${NAMESPACE}
```

### 3. Choose Scaling Method

**Ask the user for their preference or use the priority recommendations below.**

Before scaling, determine which method to use based on:
1. **User's explicit preference** (if provided)
2. **Deployment management approach** (detected from step 1)
3. **Speed vs. consistency trade-offs**
4. **Operational requirements**

#### Method Selection Guidelines

**Ask the user:**
"I've detected your deployment type. Which scaling method would you prefer?
1. **Script-based** (Recommended) - Automated, safe, handles all deployment types
2. **Helm/Helmfile** - Best for Helm-managed deployments, maintains state consistency
3. **Direct kubectl** - Fastest, but doesn't update Helm state
4. **LeaderWorkerSet** - For LWS-based deployments only

Or I can recommend the best method based on your deployment."

#### Priority Recommendations by Deployment Type

**If Helm/Helmfile-managed deployment detected:**
- **Priority 1:** Method C (Helm/Helmfile) - Maintains state consistency
- **Priority 2:** Method A (Script) - Safe fallback
- **Priority 3:** Method B (kubectl) - Only if speed is critical and user accepts state drift

**If LeaderWorkerSet deployment detected:**
- **Priority 1:** Method D (LeaderWorkerSet) - Native LWS scaling
- **Priority 2:** Method A (Script) - Can handle LWS
- **Priority 3:** Method B (kubectl) - Direct LWS scaling

**If standard Deployment detected (no Helm):**
- **Priority 1:** Method B (kubectl) - Direct and fast
- **Priority 2:** Method A (Script) - Adds validation
- **Priority 3:** Method C (Helm) - Not applicable

**If uncertain or mixed deployment:**
- **Priority 1:** Method A (Script) - Safest, handles all cases
- **Priority 2:** Ask user for clarification

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

## Complete Scaling Workflow

1. **Detect** - Run [`detect-deployment.sh`](scripts/detect-deployment.sh) to identify deployment type
2. **Validate** - Run [`check-resources.sh`](scripts/check-resources.sh) to verify available resources
3. **Choose Method** - Ask user preference or use priority recommendations (see Method Selection Guidelines)
4. **Scale** - Execute chosen method (A, B, C, or D)
5. **Monitor** - Watch pods come up and verify InferencePool discovery
6. **Test** - Verify inference routing works with new workers


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