---
name: llm-d-scale-workers
description: Execute scaling actions for llm-d prefill/decode workers on Kubernetes/OpenShift. Runs detection scripts, calculates optimal P/D ratios, and executes scaling commands directly. Supports manual scaling (immediate adjustments via existing scripts/kubectl) and automatic WVA setup (continuous autoscaling). Use when users need to handle load changes, optimize worker ratios, scale up/down for cost savings, or set up autoscaling - the skill performs the actions.
---

# llm-d Worker Scaling Skill
## 📋 Command Execution Notice

**Before executing any command, I will:**
1. **Explain what the command does** - A clear description of the command's purpose and expected outcome
2. **Show the actual command** - The exact command that will be executed
3. **Explain why it's needed** - How this command fits into the overall deployment workflow

This ensures you understand each step before it happens and can verify the actions align with your intentions.


> ## 🔔 ALWAYS NOTIFY THE USER BEFORE CREATING ANYTHING
>
> **RULE**: Before creating ANY resource — including namespaces, files or any Kubernetes object — you MUST first tell the user what you are about to create and why.
>
> **Format to use before every creation action**:
> > "I am about to create `<resource-type>` named `<name>` because `<reason>`. Proceeding now."
>
>
> **Never silently create resources.** If you are unsure whether a resource already exists, check first, then notify before acting.

## What Not To Do

Critical rules to follow when deploying and managing llm-d:

1. **Do NOT change cluster-level definitions** 
All changes must be made exclusively inside the designated project namespace. Never modify cluster-wide resources (e.g., ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes, or any resource outside the target namespace). Scope every `kubectl apply`, `helm install`, and `helmfile apply` command to the target namespace using `-n ${NAMESPACE}`.

2. **Do NOT modify any existing code you did not create** 
 Only create new files and modify them as needed. Never edit pre-existing files in the repository . If customization is required, create a new file and reference it instead.

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

## Workflow: Direct Execution Only

**CRITICAL RULES:**
1. **ALWAYS use existing scripts** from `skills/llmd-scale-workers/scripts/` directory
2. **NEVER create README.md files** - provide summaries in conversation only
3. **NEVER create new scripts** - use the existing ones
4. **Scripts run non-interactively by default** - all scripts are designed for automation

### Step 1: Detect Deployment

Execute detection immediately:
```bash
bash skills/llmd-scale-workers/scripts/detect-deployment.sh ${NAMESPACE}
```

### Step 2: Execute Scaling Action

**For WVA Autoscaling Setup:**
```bash
# Non-interactive by default - uses auto-detected values
bash skills/llmd-scale-workers/scripts/setup-wva-autoscaling.sh

# For interactive mode with prompts (optional):
INTERACTIVE=true bash skills/llmd-scale-workers/scripts/setup-wva-autoscaling.sh
```
Then provide 3-5 sentence summary in conversation.

**For Manual Scaling:**
```bash
# Non-interactive by default - scales immediately
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT}
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t prefill -r ${COUNT}

# For interactive mode with confirmation prompt (optional):
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT} -i
```
Then provide 3-5 sentence summary in conversation.

### Output Format

**Correct approach:**
- Execute the existing scripts
- Provide brief summary: "I've set up WVA autoscaling for your granite-34b deployment. It will scale between 2-10 replicas based on saturation. Monitor with: kubectl get hpa -n ${NAMESPACE}"

**WRONG - Never do this:**
- ❌ Create README.md
- ❌ Create setup-autoscaling.sh (use existing scripts/setup-wva-autoscaling.sh)
- ❌ Create monitoring-guide.md
- ❌ Create new scaling scripts (use existing scripts/scale-workers.sh)

## Scaling Methods

Choose method based on deployment type (auto-detect from Step 1):

**Preferred method by deployment type:**
- **Helm/Helmfile-managed:** Use scale-workers.sh script (maintains state)
- **LeaderWorkerSet:** Use kubectl scale leaderworkerset
- **Standard Deployment:** Use kubectl scale deployment
- **Uncertain:** Use scale-workers.sh script (safest)

**Execute scaling:**
```bash
# Using script (recommended for Helm deployments) - non-interactive by default
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT}

# Using script with interactive confirmation (optional)
bash skills/llmd-scale-workers/scripts/scale-workers.sh -n ${NAMESPACE} -t decode -r ${COUNT} -i

# Using kubectl (fast, for non-Helm)
kubectl scale deployment <name> --replicas=${COUNT} -n ${NAMESPACE}

# Using kubectl for LWS
kubectl scale leaderworkerset <name> --replicas=${COUNT} -n ${NAMESPACE}
```

**Script Modes:**
- **Default (Non-Interactive):** Scripts execute immediately without prompts - ideal for automation
- **Interactive Mode:** Use `-i` flag or `INTERACTIVE=true` environment variable for confirmation prompts


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

## Post-Scaling Verification

After scaling, verify the workers are running:
```bash
kubectl get pods -n ${NAMESPACE} -l llm-d.ai/role=decode
kubectl wait --for=condition=ready pod -l llm-d.ai/role=decode -n ${NAMESPACE} --timeout=600s
```

If issues occur, check pod status with `kubectl describe pod <pod-name> -n ${NAMESPACE}`.