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

1. **Ask the user how many workers they want to add:**
   - "How many decode workers would you like to add/scale to?"
   - "How many prefill workers would you like to add/scale to?"

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

### 3. Scale Workers

**Method A: Using Script (Recommended)**
```bash
# Scale decode workers (replace ${DECODE_COUNT} with user's desired count)
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t decode -r ${DECODE_COUNT}

# Scale prefill workers (replace ${PREFILL_COUNT} with user's desired count)
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t prefill -r ${PREFILL_COUNT}
```

**Method B: Direct kubectl**
```bash
# Quick scale
kubectl scale deployment <deployment-name> --replicas=3 -n ${NAMESPACE}
```

**Method C: Helm/Helmfile (for Helm-managed deployments)**
```bash
# 1. Edit values file (e.g., guides/pd-disaggregation/ms-pd/values.yaml)
#    Change: decode.replicas: 1 → 3

# 2. Apply changes
cd guides/<guide-name>
helmfile apply -n ${NAMESPACE}
```

**Method D: LeaderWorkerSet**
```bash
kubectl scale leaderworkerset <lws-name> --replicas=3 -n ${NAMESPACE}
```

## Scaling Workflow

1. **Detect** - Run [`detect-deployment.sh`](scripts/detect-deployment.sh) to identify deployment type
2. **Validate** - Run [`check-resources.sh`](scripts/check-resources.sh) to verify available resources
3. **Scale** - Use [`scale-workers.sh`](scripts/scale-workers.sh) or direct kubectl/helm
4. **Monitor** - Watch pods come up and verify InferencePool discovery
5. **Test** - Verify inference routing works with new workers

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

## Examples

### Example 1: Interactive Scaling Workflow
```
User: "I want to scale my workers"

Agent: "How many decode workers would you like to add/scale to?"
User: "5 decode workers"

Agent: "How many prefill workers would you like to add/scale to?"
User: "10 prefill workers"

Agent: [Runs check-resources.sh]
Agent: "Your cluster has 32 available GPUs. With TP=4 for decode (4 GPUs each) and TP=1 for prefill (1 GPU each):
- 5 decode workers need: 20 GPUs
- 10 prefill workers need: 10 GPUs
- Total needed: 30 GPUs ✓ Available

This gives you a 2:1 prefill:decode ratio, which is good for balanced workloads.
Shall I proceed with scaling to 5 decode and 10 prefill workers?"

User: "Yes"
Agent: [Executes scaling commands]
```

### Example 2: Resource-Constrained Scenario
```
User: "Add 10 decode workers"

Agent: [Runs check-resources.sh]
Agent: "Your cluster has 16 available GPUs. With TP=4 for decode workers (4 GPUs each):
- 10 decode workers would need: 40 GPUs ❌ Not available
- Maximum possible: 4 decode workers (16 GPUs)

Would you like to proceed with 4 decode workers instead?"
```

### Example 3: Unreasonable Ratio Suggestion
```
User: "I want 1 decode worker and 20 prefill workers"

Agent: "I notice you're requesting a 20:1 prefill:decode ratio. This ratio is typically used for very high input/low output workloads (like 10k+ input tokens, <1k output tokens).

What is your expected workload pattern?
- High input/low output (10k ISL / 1k OSL)
- High output/low input (1k ISL / 10k OSL)
- Balanced (4k ISL / 4k OSL)

Based on your answer, I can recommend a more optimal ratio."
```

### Example 4: Direct Scaling Commands
```bash
# After user confirms, scale decode workers to their requested count
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t decode -r ${USER_REQUESTED_DECODE_COUNT}

# Scale prefill workers to their requested count
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t prefill -r ${USER_REQUESTED_PREFILL_COUNT}
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