---
name: llm-d-kubernetes-deployment
description: Configure and deploy llm-d (high-performance distributed LLM inference) on an existing Kubernetes clusters using Well-lit Path guides for production-ready LLM serving with optimizations like intelligent inference scheduling, prefill/decode disaggregation, wide expert parallelism, and tiered prefix caching.
---

# llm-d Kubernetes Deployment Skill

> ## 🔔 ALWAYS NOTIFY THE USER BEFORE CREATING ANYTHING
>
> **RULE**: Before creating ANY resource — including namespaces, PVCs, files, Helm releases, HTTPRoutes, or any Kubernetes object — you MUST first tell the user what you are about to create and why.
>
> **Format to use before every creation action**:
> > "I am about to create `<resource-type>` named `<name>` because `<reason>`. Proceeding now."
>
> **Examples**:
> - "I am about to create namespace `llm-d` because it does not exist yet. Proceeding now."
> - "I am about to create PVC `llm-d-kv-cache-storage` (18000Gi, storage class: default) because no PVC was found in namespace `llm-d`. Proceeding now."
> - "I am about to create file `${LLMD_PATH}/pvc.yaml` with the PVC definition. Proceeding now."
> - "I am about to apply the HTTPRoute from `httproute.yaml` to namespace `llm-d`. Proceeding now."
>
> **Never silently create resources.** If you are unsure whether a resource already exists, check first, then notify before acting.

This skill enables AI agents to guide users through deploying llm-d on Kubernetes clusters using Well-lit Path guides. llm-d provides tested and benchmarked recipes for serving large language models at peak performance with production best practices.

## Overview

llm-d offers four main Well-lit Path deployment strategies:

1. **Intelligent Inference Scheduling** (`guides/inference-scheduling/`) - Default deployment with load-aware and prefix-cache aware routing
2. **Prefill/Decode Disaggregation** (`guides/pd-disaggregation/`) - Split inference for large models with long prompts
3. **Wide Expert Parallelism** (`guides/wide-ep-lws/`) - Deploy very large MoE models with Data/Expert Parallelism
4. **Tiered Prefix Cache** (`guides/tiered-prefix-cache/`) - Extend cache capacity beyond accelerator memory

## When to Use This Skill

Activate this skill when users need to:
- Deploy LLM inference on Kubernetes
- Optimize LLM serving performance
- Set up production-ready model serving infrastructure
- Choose between different deployment strategies
- Configure hardware-specific deployments (NVIDIA, AMD, Intel, TPU, CPU)

## Prerequisites

Each Well-lit Path guide contains its own prerequisites section. Common prerequisites include:

1. **LLMD_PATH Environment Variable** - Must point to your llm-d repository clone
2. **Client Tools** - kubectl, helm, helmfile, git (see `guides/prereq/client-setup/README.md`)
3. **HuggingFace Token** - Secret `llm-d-hf-token` with key `HF_TOKEN` in target namespace
4. **Gateway Provider** - Istio, K-Gateway, or Agent Gateway (see `guides/prereq/gateway-provider/README.md`)
5. **Infrastructure** - Sufficient cluster resources (see `guides/prereq/infrastructure/README.md`)
6. **Monitoring** (optional) - Prometheus and Grafana (see `docs/monitoring/README.md`)

**Always refer to the specific guide's README.md for detailed prerequisites.**

## Deployment Workflow

When a user requests llm-d deployment, follow this workflow:

### Step 1: Select Well-lit Path Guide

**Ask the user which deployment strategy they want to use:**

Present the four options with brief descriptions:
1. **Intelligent Inference Scheduling** - Best for most deployments (default, load-aware routing)
2. **Prefill/Decode Disaggregation** - For large models with long prompts (requires RDMA)
3. **Wide Expert Parallelism** - For very large MoE models like DeepSeek-R1 (requires 32+ GPUs, RDMA)
4. **Tiered Prefix Cache** - Add to any path for extended cache capacity

**Default suggestion**: Intelligent Inference Scheduling (unless user has specific requirements)

### Step 2: Gather Project Configuration

**Ask for the namespace/project name:**
- Default: `llm-d` (short names recommended to avoid hostname length issues)
- Ask: "What Kubernetes namespace would you like to use?"

**Ask for release name postfix (optional):**
- Default: Empty (no postfix)
- Ask: "Do you need a custom release name postfix for concurrent installations?" *(optional)*

### Step 3: Read Guide Configuration and Detect Defaults

**Read the selected guide's README.md** from `guides/{selected-path}/README.md` to understand:
- Default configuration values
- Hardware requirements
- Prerequisites
- Deployment commands

**Explore existing namespace resources:**
```bash
# Check if namespace exists
kubectl get namespace ${NAMESPACE}

# Check for existing resources in namespace (if it exists)
kubectl get all,pvc,httproute,gateway,inferencepool -n ${NAMESPACE}

# Check for HuggingFace token secret
kubectl get secret llm-d-hf-token -n ${NAMESPACE}
```

**Detect and present defaults:**

Based on the guide's configuration files (e.g., `values.yaml`, `helmfile.yaml`), present detected defaults:
- Hardware backend (from guide defaults, typically `cuda`)
- Gateway provider (from guide defaults, typically `istio`)
- Storage class (if PVC needed, typically `default`)
- Model configuration (replica count, GPU allocation, etc.)

**Ask: "The guide uses these defaults: [list defaults]. Do you want to change any of these?"**

Only if the user wants to change defaults, ask for specific values. Otherwise, proceed with detected defaults.

### Step 4: Follow the Guide's Deployment Steps

**Navigate to the guide directory:**
```bash
cd ${LLMD_PATH}/guides/{selected-path}
```

**Follow the guide's README.md deployment instructions exactly.** The guide contains:
- Prerequisites checklist
- Exact deployment commands
- Configuration options
- Validation steps

**Key principle**: Mimic the manual experience - only provide input when changing defaults.

#### Common Deployment Pattern

Most guides follow this pattern (refer to the specific guide for exact commands):

1. **Check/Create namespace:**
   ```bash
   kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
   ```

2. **Verify prerequisites** (as listed in guide's README.md):
   - Client tools installed
   - HuggingFace token secret created
   - Gateway provider deployed
   - Infrastructure requirements met

3. **Deploy using helmfile** (command from guide):
   ```bash
   helmfile apply -n ${NAMESPACE}
   # Add -e flags only if changing defaults: -e <hardware> -e <gateway>
   ```

4. **Apply HTTPRoute** (if guide includes it):
   ```bash
   kubectl apply -f httproute.yaml -n ${NAMESPACE}
   # Or httproute.gke.yaml for GKE
   ```

5. **Validate deployment** (commands from guide):
   ```bash
   kubectl get pods -n ${NAMESPACE}
   kubectl get httproute,gateway,inferencepool -n ${NAMESPACE}
   ```

### Step 5: Execute the Deployment Plan

**Execute ALL commands from the guide sequentially.** Do NOT just show commands — run them one by one, observe output, and fix any errors before proceeding.

#### Execution Rules

1. **Run every command** from the guide in order using `execute_command`
2. **After each command**, inspect the output:
   - Success → proceed to next command
   - Failure → diagnose error, apply fix, re-run before continuing
3. **Never skip a step** — each depends on the previous one succeeding
4. **Always use LLMD_PATH** environment variable when navigating directories

#### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `namespace already exists` | Previously created | Safe to ignore; continue |
| `Error: INSTALLATION FAILED: cannot re-use a name` | Helm release exists | Use `helmfile apply` (idempotent) |
| `error: the server doesn't have a resource type "inferencepool"` | Gateway API CRDs missing | Install Gateway API CRDs |
| `Pending` pods / `0/N nodes available` | Insufficient resources | Check node labels, tolerations, GPU capacity |
| `CrashLoopBackOff` | Container startup failure | Run `kubectl logs <pod> -n ${NAMESPACE}` and `kubectl describe pod <pod>` |
| `ImagePullBackOff` | Image not accessible | Verify registry credentials and image name/tag |

### Step 6: Validate Deployment

Follow the guide's validation steps. Typically:

```bash
# Check pod status
kubectl get pods -n ${NAMESPACE}

# Verify resources
kubectl get httproute,gateway,inferencepool -n ${NAMESPACE}

# Check helm releases
helm list -n ${NAMESPACE}
```

If any resource is not in expected state, diagnose using:
```bash
kubectl describe <resource-type> <resource-name> -n ${NAMESPACE}
kubectl logs <pod-name> -n ${NAMESPACE}
```

### Step 7: Provide Next Steps

Point user to guide's documentation for:
- Testing inference endpoint
- Running benchmarks (if applicable)
- Monitoring the deployment
- Optimization recommendations

## Common Issues and Solutions

### Long Namespace Names
**Issue**: Generated pod hostnames become too long (>64 characters)
**Solution**: Use shorter namespace names (e.g., `llm-d`) and set `RELEASE_NAME_POSTFIX` environment variable

### RDMA Connectivity Failures (Wide-EP)
**Issue**: All-to-All RDMA connectivity not working
**Solution**: Ensure every NIC can communicate with every NIC on all hosts (not just rail-only connectivity)

### Pod Scheduling Failures
**Issue**: Pods not scheduling
**Solution**: Check cluster resources, node selectors, and tolerations. Verify hardware requirements are met

### Gateway Routing Not Working
**Issue**: HTTPRoute not routing traffic
**Solution**: Verify gateway provider is correctly deployed and HTTPRoute resources are created. Check gateway configuration

## Additional Resources

- **Quickstart**: `guides/QUICKSTART.md`
- **Project Overview**: `PROJECT.md`
- **Contributing**: `CONTRIBUTING.md`
- **Gateway Customization**: `docs/customizing-your-gateway.md`
- **vLLM Documentation**: https://docs.vllm.ai
- **Inference Gateway**: https://github.com/kubernetes-sigs/gateway-api-inference-extension
- **Inference Scheduler Architecture**: https://github.com/llm-d/llm-d-inference-scheduler/blob/main/docs/architecture.md

## Security Considerations

- Run deployments in isolated namespaces
- Use RBAC for access control
- Secure HuggingFace tokens as Kubernetes secrets
- Enable network policies for pod-to-pod communication
- Use TLS for gateway ingress
- Audit all deployments and changes

## Combining Well-lit Paths

Tiered Prefix Cache can be combined with any of the other three Well-lit Paths:
- Inference Scheduling + Tiered Cache
- P/D Disaggregation + Tiered Cache
- Wide-EP + Tiered Cache

This provides both performance optimization and extended cache capacity.

## What Not To Do

Critical rules to follow when deploying and managing llm-d:

1. **Do NOT change cluster-level definitions** — All changes must be made exclusively inside the designated project namespace. Never modify cluster-wide resources (e.g., ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes, or any resource outside the target namespace). Scope every `kubectl apply`, `helm install`, and `helmfile apply` command to the target namespace using `-n ${NAMESPACE}`.

2. **Do NOT modify any existing code you did not create** — Only create new files and modify them as needed. Never edit pre-existing files in the repository (e.g., existing `values.yaml`, `helmfile.yaml`, `httproute.yaml`, `README.md`, or any other committed file). If customization is required, create a new file (e.g., `values-custom.yaml`, `httproute-custom.yaml`) and reference it instead.