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
>
> **Never silently create resources.** If you are unsure whether a resource already exists, check first, then notify before acting.

This skill enables AI agents to guide users through deploying llm-d on Kubernetes clusters using Well-lit Path guides. llm-d provides tested and benchmarked recipes for serving large language models at peak performance with production best practices.

## Overview

llm-d provides Well-lit Path deployment guides located in the `guides/` directory. Each guide has a README.md file with the prefix "Well-lit Path:" in its title, containing tested and benchmarked recipes for specific deployment strategies.

**To discover available Well-lit Paths**, the agent will:
1. List directories under `guides/` (excluding `prereq/` and `recipes/`)
2. Check each directory for a README.md file
3. Read the README.md title to identify Well-lit Path guides (those starting with "Well-lit Path:")
4. Present the available guides to the user with their descriptions

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

### Step 0: Verify LLMD_PATH or Get Repository Location

Use LLMD_PATH environment variable if set; if not set, ask the user for the llm-d repository path.

### Step 1: Discover and Select Well-lit Path Guide

**Automatically discover available Well-lit Path guides:**

1. **List guide directories:**
   ```bash
   ls -d ${LLMD_PATH}/guides/*/ | grep -v -E '(prereq|recipes)' | xargs -n1 basename
   ```

2. **For each directory, check if it contains a Well-lit Path guide:**
   ```bash
   # Check if README.md exists and starts with "Well-lit Path:"
   head -n 5 ${LLMD_PATH}/guides/{guide-dir}/README.md | grep "^# Well-lit Path:"
   ```

3. **Extract guide information:**
   - Guide name (from README.md title after "Well-lit Path:")
   - Guide directory name
   - Brief description (from Overview section)

**Present discovered guides to user:**

"Available Well-lit Path deployment strategies:
1. {Guide 1 Name} (`guides/{dir1}/`) - {brief description}
2. {Guide 2 Name} (`guides/{dir2}/`) - {brief description}
...

Which deployment strategy would you like to use?"

**Default suggestion**: If `inference-scheduling` guide exists, suggest it as default for most deployments.

### Step 2: Auto-Detect Current Project/Namespace

1. **For OpenShift clusters**, check current project:
   ```bash
   oc project
   ```
   This returns the current project name, which is also the namespace.

2. **For standard Kubernetes**, check current context namespace:
   ```bash
   kubectl config view --minify --output 'jsonpath={..namespace}'
   ```

3. **If no namespace is detected**, only then suggest default: `llm-d`

**Present detected namespace to user:**
- If detected: "Detected current namespace: `{detected-namespace}`. Using this for deployment."
- If not detected: "No current namespace detected. Would you like to use `llm-d`?"

**Auto-detect release name postfix:**
- Check for existing releases in the namespace to detect if postfix is needed
- Only ask if concurrent installations are detected or if detection fails

### Step 3: Auto-Detect Configuration and Resources

**Read the selected guide's README.md** from `guides/{selected-path}/README.md` to understand requirements.

**Automatically detect existing resources and configuration:**

1. **Detect Hardware/Accelerators:**
   ```bash
   # Check for NVIDIA GPUs
   kubectl get nodes -o json | jq '.items[].status.capacity | select(."nvidia.com/gpu" != null)'
   
   # Check for AMD GPUs
   kubectl get nodes -o json | jq '.items[].status.capacity | select(."amd.com/gpu" != null)'
   
   # Check for Intel XPU
   kubectl get nodes -o json | jq '.items[].status.capacity | select(."gpu.intel.com/i915" != null)'
   
   # Check for Intel Gaudi (HPU)
   kubectl get nodes -o json | jq '.items[].status.capacity | select(."habana.ai/gaudi" != null)'
   
   # Check for TPU (GKE)
   kubectl get nodes -o json | jq '.items[].status.capacity | select(."google.com/tpu" != null)'
   ```
   
   Based on detection, set hardware backend: `cuda`, `amd`, `xpu`, `hpu`, `tpu`, or `cpu` (if no accelerators found)

2. **Detect Gateway Provider:**
   ```bash
   # Check for Istio
   kubectl get namespace istio-system 2>/dev/null && echo "istio"
   
   # Check for K-Gateway
   kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null && kubectl get pods -n kube-system -l app=gateway 2>/dev/null && echo "kgateway"
   
   # Check for Agent Gateway
   kubectl get deployment -n gateway-system agent-gateway 2>/dev/null && echo "agentgateway"
   
   # Check for GKE Gateway (check if running on GKE)
   kubectl get nodes -o json | jq -r '.items[0].metadata.labels["cloud.google.com/gke-nodepool"]' 2>/dev/null && echo "gke"
   ```

3. **Detect Storage Class:**
   ```bash
   # Get default storage class
   kubectl get storageclass -o json | jq -r '.items[] | select(.metadata.annotations["storageclass.kubernetes.io/is-default-class"]=="true") | .metadata.name'
   
   # If no default, list available storage classes
   kubectl get storageclass -o name
   ```

4. **Check existing resources in namespace:**
   ```bash
   # Check if namespace exists
   kubectl get namespace ${NAMESPACE} 2>/dev/null
   
   # If exists, check for existing resources
   kubectl get all,pvc,httproute,gateway,inferencepool -n ${NAMESPACE} 2>/dev/null
   
   # Check for HuggingFace token secret
   kubectl get secret llm-d-hf-token -n ${NAMESPACE} 2>/dev/null
   ```

**Present detected configuration:**

"Detected configuration:
- Namespace: `{detected-namespace}`
- Hardware: `{detected-hardware}` ({count} nodes with {accelerator-type})
- Gateway Provider: `{detected-gateway}`
- Storage Class: `{detected-storage-class}`
- Existing resources: {list if any found}"

**Only ask user for input if:**
1. Auto-detection failed for any component
2. Detected resource doesn't exist or is invalid
3. User wants to override detected values

**Ask: "Proceed with detected configuration, or would you like to change anything?"**

If user wants changes, ask specifically for the values they want to override. Otherwise, proceed with detected configuration.

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

**Key principle**: Only provide input when changing defaults.

#### Common Deployment Pattern

Most guides follow this pattern (refer to the specific guide for exact commands):

1. **Check/Create namespace:**
   ```bash
   kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
   ```

2. **Auto-detect or create PVC (if needed by the guide):**
   
   First, check if a PVC already exists in the namespace:
   ```bash
   kubectl get pvc -n ${NAMESPACE}
   ```
   
   - **If PVC exists and is Bound**: Use the existing PVC. Inform user: "Found existing PVC `{pvc-name}` in namespace `{namespace}`. Using it for deployment."
   
   - **If no PVC exists**: Check the guide to determine if a PVC is required. If required:
     - Read PVC configuration from guide (typically in `manifests/pvc.yaml` or similar)
     - Detect storage class (from Step 3 auto-detection)
     - Inform user: "No PVC found in namespace `{namespace}`. Creating PVC `{pvc-name}` with storage class `{storage-class}` and size `{size}`."
     - Create and apply the PVC:
       ```bash
       kubectl apply -f {guide-path}/manifests/pvc.yaml -n ${NAMESPACE}
       # Or create inline if guide doesn't have a PVC manifest
       ```
     - Wait for PVC to be Bound before proceeding:
       ```bash
       kubectl wait --for=condition=Bound pvc/{pvc-name} -n ${NAMESPACE} --timeout=300s
       ```

3. **Verify prerequisites** (as listed in guide's README.md):
   - Client tools installed
   - HuggingFace token secret created
   - Gateway provider deployed
   - Infrastructure requirements met
   - PVC is Bound (if required)

4. **Deploy using helmfile** (command from guide):
   ```bash
   helmfile apply -n ${NAMESPACE}
   # Add -e flags only if changing defaults: -e <hardware> -e <gateway>
   ```

5. **Apply HTTPRoute** (if guide includes it):
   ```bash
   kubectl apply -f httproute.yaml -n ${NAMESPACE}
   # Or httproute.gke.yaml for GKE
   ```

6. **Validate deployment** (commands from guide):
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
- **Architecture**: `https://llm-d.ai/docs/architecture`
- **Gateway Customization**: `docs/customizing-your-gateway.md`
- **Inference Gateway**: https://github.com/kubernetes-sigs/gateway-api-inference-extension
- **Inference Scheduler Architecture**: https://github.com/llm-d/llm-d-inference-scheduler/blob/main/docs/architecture.md

## Security Considerations

- Run deployments in isolated namespaces
- Use RBAC for access control
- Secure HuggingFace tokens as Kubernetes secrets
- Enable network policies for pod-to-pod communication
- Use TLS for gateway ingress
- Audit all deployments and changes

## What Not To Do

Critical rules to follow when deploying and managing llm-d:

1. **Do NOT change cluster-level definitions** — All changes must be made exclusively inside the designated project namespace. Never modify cluster-wide resources (e.g., ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes, or any resource outside the target namespace). Scope every `kubectl apply`, `helm install`, and `helmfile apply` command to the target namespace using `-n ${NAMESPACE}`.

2. **Do NOT modify any existing code you did not create** — Only create new files and modify them as needed. Never edit pre-existing files in the repository (e.g., existing `values.yaml`, `helmfile.yaml`, `httproute.yaml`, `README.md`, or any other committed file). If customization is required, create a new file (e.g., `values-custom.yaml`, `httproute-custom.yaml`) and reference it instead.