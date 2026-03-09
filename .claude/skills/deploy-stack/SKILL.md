---
name: llm-d-kubernetes-deployment
description: Configure and deploy llm-d (high-performance distributed LLM inference) on an existing Kubernetes clusters using Well-lit Path guides for production-ready LLM serving with optimizations like intelligent inference scheduling, prefill/decode disaggregation, wide expert parallelism, and tiered prefix caching.
---

# llm-d Kubernetes Deployment Skill
## 📋 Command Execution Notice

**Before executing any command, I will:**
1. **Explain what the command does** - A clear description of the command's purpose and expected outcome
2. **Show the actual command** - The exact command that will be executed
3. **Explain why it's needed** - How this command fits into the overall deployment workflow

This ensures you understand each step before it happens and can verify the actions align with your intentions.


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

1. **LLMD_PATH Environment Variable** - Point to your llm-d repository clone
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

**Default suggestion**: If `inference-scheduling` guide exists, suggest it as default.

### Step 2: Auto-Detect Current Project/Namespace

1. check if a NAMESPACE environment variable is specified. 
2. check if an oc project exists.
3. If none of the above holds, ask the user for the NAMESPACE where it is deployed.
4. Make sure the NAMESPACE environment variable is set.

**Auto-detect release name postfix:**
- Check for existing releases in the namespace to detect if postfix is needed
- Only ask if concurrent installations are detected or if detection fails

### Step 3: Auto-Detect Configuration and Resources

**Extract requirements from the guide:**
- Parse the guide's README.md to identify required components e.g., Gateway API, Kueue, monitoring tools
- Identify hardware requirements (GPU types, memory, storage)
- Note any prerequisite configurations or dependencies
- Extract recommended values and configuration parameters

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

If user wants changes, ask specifically for the values they want to override. Otherwise, proceed with detected configuration.

### Step 4: Execute Deployment Actions

**Navigate to the guide directory:**
```bash
cd ${LLMD_PATH}/guides/{selected-path}
```

**Key principle**: Only provide input when changing defaults. Use auto-detected values unless user explicitly overrides.

#### Deployment Actions

Execute these actions in sequence, adapting based on the specific guide's requirements:

#### 4.1 Namespace Management

**Verify namespace exists:**
```bash
kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
```

#### 4.2 Storage Provisioning

**Check for existing PVC:**
```bash
kubectl get pvc -n ${NAMESPACE}
```

**If PVC is required by the guide:**
- **PVC exists and is Bound**: Use existing PVC. Inform user: "Found existing PVC `{pvc-name}` in namespace `{namespace}`. Using it for deployment."
- **No PVC exists**:
  - Read PVC configuration from guide (typically in `manifests/pvc.yaml`)
  - Use auto-detected storage class from Step 3
  - Inform user: "Creating PVC `{pvc-name}` with storage class `{storage-class}` and size `{size}`."
  - Apply PVC manifest:
    ```bash
    kubectl apply -f {guide-path}/manifests/pvc.yaml -n ${NAMESPACE}
    ```
  - Wait for PVC to be Bound:
    ```bash
    kubectl wait --for=condition=Bound pvc/{pvc-name} -n ${NAMESPACE} --timeout=300s
    ```

#### 4.3 Prerequisites Verification

**Verify all prerequisites are met:**
- Client tools installed (kubectl, helm, helmfile)
- HuggingFace token secret created (if required by guide)
- Gateway provider deployed (if required by guide)
- Infrastructure requirements met (nodes, accelerators)
- PVC is Bound (if required by guide)

**Check each prerequisite programmatically:**
```bash
# Example: Check for HuggingFace secret
kubectl get secret hf-token -n ${NAMESPACE} 2>/dev/null || echo "Warning: HuggingFace token secret not found"

# Example: Check for Gateway API CRDs
kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null || echo "Warning: Gateway API CRDs not installed"
```

#### 4.4 Helmfile Deployment

**Deploy using helmfile with auto-detected configuration:**
```bash
helmfile apply -n ${NAMESPACE}
```

**Only add environment flags if overriding defaults:**
- Hardware override: `-e <hardware>` (e.g., `-e cuda`, `-e tpu`)
- Gateway override: `-e <gateway>` (e.g., `-e istio`, `-e kgateway`)

**Example with overrides:**
```bash
helmfile apply -n ${NAMESPACE} -e cuda -e istio
```

#### 4.5 HTTPRoute Configuration

**If guide includes HTTPRoute, apply it:**
```bash
# Standard HTTPRoute
kubectl apply -f httproute.yaml -n ${NAMESPACE}

# Or GKE-specific HTTPRoute
kubectl apply -f httproute.gke.yaml -n ${NAMESPACE}
```

#### 4.6 Deployment Validation

**Verify resources are created:**
```bash
kubectl get pods -n ${NAMESPACE}
kubectl get httproute,gateway,inferencepool -n ${NAMESPACE}
```

**Check pod status:**
```bash
kubectl get pods -n ${NAMESPACE} -w
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

Execute these validation steps to ensure successful deployment:

#### 6.1 Pod Status Validation

**Check all pods are running:**
```bash
kubectl get pods -n ${NAMESPACE}
```

**Expected states:**
- All pods should be in `Running` state
- Ready column should show `N/N` (all containers ready)
- Restarts should be 0 or minimal

**If pods are not running, check status:**
```bash
# Pending pods - check events
kubectl describe pod <pod-name> -n ${NAMESPACE}

# CrashLoopBackOff - check logs
kubectl logs <pod-name> -n ${NAMESPACE}
kubectl logs <pod-name> -n ${NAMESPACE} --previous

# ImagePullBackOff - verify image and credentials
kubectl describe pod <pod-name> -n ${NAMESPACE} | grep -A 5 "Events:"
```

#### 6.2 Resource Validation

**Verify all expected resources exist:**
```bash
# Check InferencePool
kubectl get inferencepool -n ${NAMESPACE}

# Check Gateway and HTTPRoute
kubectl get gateway,httproute -n ${NAMESPACE}

# Check Services
kubectl get svc -n ${NAMESPACE}

# Check PVCs (if applicable)
kubectl get pvc -n ${NAMESPACE}
```

**Verify resource status:**
```bash
# InferencePool should show Ready status
kubectl describe inferencepool -n ${NAMESPACE}

# Gateway should be Programmed and Accepted
kubectl describe gateway -n ${NAMESPACE}

# HTTPRoute should be Accepted
kubectl describe httproute -n ${NAMESPACE}
```

#### 6.3 Helm Release Validation

**Verify Helm releases are deployed:**
```bash
helm list -n ${NAMESPACE}
```

**Expected output:**
- All releases should show `STATUS: deployed`
- Chart versions should match expected versions

#### 6.4 Connectivity Validation

**Test internal connectivity:**
```bash
# Get service endpoints
kubectl get endpoints -n ${NAMESPACE}

# Test service DNS resolution from a test pod
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n ${NAMESPACE} -- sh
# Inside pod: curl http://<service-name>.<namespace>.svc.cluster.local/health
```

**Test external access (if HTTPRoute is configured):**
```bash
# Get Gateway address
kubectl get gateway -n ${NAMESPACE} -o jsonpath='{.items[0].status.addresses[0].value}'

# Test endpoint (replace with actual gateway address)
curl http://<gateway-address>/v1/models
```

#### 6.5 Logs Validation

**Check for errors in logs:**
```bash
# Check all pod logs for errors
kubectl logs -l app=<app-label> -n ${NAMESPACE} --tail=50

# Check specific components
kubectl logs -l component=vllm -n ${NAMESPACE} --tail=100
kubectl logs -l component=gateway -n ${NAMESPACE} --tail=100
```

**Look for:**
- Successful model loading messages
- No error or warning messages
- Healthy startup sequences

#### 6.6 Performance Validation

**Verify resource utilization:**
```bash
# Check CPU and memory usage
kubectl top pods -n ${NAMESPACE}

# Check GPU utilization (if applicable)
kubectl exec -it <pod-name> -n ${NAMESPACE} -- nvidia-smi
```

**Expected behavior:**
- Pods should not be at resource limits
- GPUs should be allocated and visible
- Memory usage should be stable

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