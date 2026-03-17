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

## Quick Start

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
# Scale decode workers
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t decode -r 3

# Scale prefill workers
bash .claude/skills/llmd-scale-workers/scripts/scale-workers.sh \
  -n ${NAMESPACE} -t prefill -r 8
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

**High Input/Low Output (10k ISL / 1k OSL)**
- Ratio: 4:1 or 8:1 (prefill:decode)
- More prefill workers needed

**Low Input/High Output (1k ISL / 10k OSL)**
- Ratio: 1:2 or 1:4 (prefill:decode)
- More decode workers needed

**Balanced (4k ISL / 4k OSL)**
- Ratio: 2:1 or 1:1 (prefill:decode)

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

### Example 1: Scale Decode Workers (Helm)
```bash
# Current: 1 decode, 4 prefill
# Goal: 3 decode workers

cd ${LLMD_PATH}/guides/pd-disaggregation
# Edit ms-pd/values.yaml: decode.replicas: 3
helmfile apply -n ${NAMESPACE}
```

### Example 2: Quick Scale with kubectl
```bash
# Scale decode to 3
kubectl scale deployment ms-pd-decode --replicas=3 -n llmd-ns

# Verify
kubectl get pods -n llmd-ns -l llm-d.ai/role=decode
```

### Example 3: Scale Both Workers
```bash
# Scale decode and prefill
bash scripts/scale-workers.sh -n llmd-ns -t decode -r 3
bash scripts/scale-workers.sh -n llmd-ns -t prefill -r 8
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