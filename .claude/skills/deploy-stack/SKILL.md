---
name: llm-d-kubernetes-deployment
description: Deploy and configure llm-d (high-performance distributed LLM inference) on Kubernetes clusters using Well-lit Path guides for production-ready LLM serving with optimizations like intelligent inference scheduling, prefill/decode disaggregation, wide expert parallelism, and tiered prefix caching.
---

# llm-d Kubernetes Deployment Skill

> ## ⚠️ ALWAYS ASK THE USER FIRST — BEFORE RUNNING ANY COMMANDS
>
> **IMPORTANT**: Before checking any local tooling, running any commands, or taking any action, you MUST ask the user for all required configuration values. Do NOT assume, auto-detect, or silently use defaults. Always present the options and suggest defaults explicitly, then wait for the user's confirmation.
>
> **Required questions to ask BEFORE proceeding**:
> 1. **Namespace**: "What Kubernetes namespace would you like to use?" *(default: `llm-d`)*
> 2. **Hardware accelerator**: "What hardware are you using — NVIDIA GPU, AMD GPU, Intel XPU, Intel Gaudi (HPU), TPU, or CPU-only?" *(default: NVIDIA CUDA)*
> 3. **Gateway provider**: "Which gateway provider would you like to use — Istio, K-Gateway, Agent Gateway, GKE, or DigitalOcean?" *(default: Istio)*
> 4. **Release name postfix** (optional): "Do you need a custom release name postfix for concurrent installs?" *(default: none)*
> 5. **Storage class** (if Tiered Cache or benchmarks): "What storage class should be used for the PVC?" *(default: `default`)*
>
> Only after the user has answered (or explicitly accepted the defaults) should you proceed with any tooling checks or command execution.

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

This skill enables IBM Bob to guide users through deploying llm-d on Kubernetes clusters using the Well-lit Path guides. llm-d provides tested and benchmarked recipes for serving large language models at peak performance with production best practices.

## Overview

llm-d offers four main Well-lit Path deployment strategies, each optimized for specific use cases:

1. **Intelligent Inference Scheduling** - Default deployment with load-aware and prefix-cache aware routing
2. **Prefill/Decode Disaggregation** - Split inference for large models with long prompts
3. **Wide Expert Parallelism** - Deploy very large MoE models with Data/Expert Parallelism
4. **Tiered Prefix Cache** - Extend cache capacity beyond accelerator memory

## When to Use This Skill

Activate this skill when users need to:
- Deploy LLM inference on Kubernetes
- Optimize LLM serving performance
- Set up production-ready model serving infrastructure
- Choose between different deployment strategies
- Configure hardware-specific deployments (NVIDIA, AMD, Intel, TPU, CPU)

## Well-lit Path Guides

All Well-lit Path guides are located in the `guides/` directory. Each guide has a README.md file with the prefix "Well-lit Path:" in its title.

### 1. Intelligent Inference Scheduling

**Location**: `guides/inference-scheduling/README.md`

**Use Cases**:
- Most vLLM deployments requiring optimized scheduling
- Workloads needing reduced tail latency
- Deployments requiring load-aware balancing

**Hardware Requirements**:
- Minimum: 2 GPUs (any supported type)
- Recommended: 16 GPUs (8 replicas x 2 GPUs each)
- Supported: NVIDIA, AMD, Intel XPU, Intel Gaudi (HPU), TPU, CPU-only

**Key Features**:
- Load-aware and prefix-cache aware balancing
- Approximate prefix cache aware scorer
- Multiple gateway provider support (Istio, K-Gateway, Agent Gateway)

**Deployment Command**:
```bash
export NAMESPACE=llm-d-inference-scheduler
kubectl create namespace ${NAMESPACE}
cd ${LLMD_PATH}/guides/inference-scheduling
helmfile apply -n ${NAMESPACE}
```

### 2. Prefill/Decode Disaggregation

**Location**: `guides/pd-disaggregation/README.md`

**Use Cases**:
- Large models (e.g., Llama-70B, GPT-OSS-120B+)
- Processing very long prompts (e.g., 10k ISL | 1k OSL)
- Sparse MoE architectures with wide-EP opportunities
- Workloads requiring lower inter-token latency

**Hardware Requirements**:
- Minimum: 8 NVIDIA GPUs
- Networking: RDMA via InfiniBand or RoCE required
- Validated on: 8xH200 clusters, Intel Gaudi2/3

**Key Features**:
- Heterogeneous parallelism (P workers with less parallelism, D workers with more)
- Tunable xPyD ratios for ISL|OSL optimization
- NIXL-based KV cache transfer

**Best Practices**:
- Use RDMA for KV cache transfer
- Deploy P workers with less parallelism and more replicas
- Deploy D workers with more parallelism and fewer replicas
- Tune P:D worker ratios based on ISL:OSL ratio

**Deployment Command**:
```bash
export NAMESPACE=llm-d-pd
kubectl create namespace ${NAMESPACE}
cd ${LLMD_PATH}/guides/pd-disaggregation
helmfile apply -n ${NAMESPACE}
```

### 3. Wide Expert Parallelism

**Location**: `guides/wide-ep-lws/README.md`

**Use Cases**:
- Very large MoE models (e.g., DeepSeek-R1)
- Workloads requiring maximum throughput
- Deployments with high-speed inter-accelerator networking

**Hardware Requirements**:
- Minimum: 32 NVIDIA H200 or B200 GPUs
- Networking: InfiniBand or RoCE RDMA with full mesh connectivity
- Special: All-to-All RDMA connectivity required (every NIC must communicate with every NIC)
- Controller: LeaderWorkerSet must be deployed

**Key Features**:
- P/D disaggregation with NIXL
- Wide expert parallelism pattern
- LeaderWorkerSet orchestration
- Support for DeepSeek-R1-0528 and similar models

**Deployment Pattern**:
- 1 DP=16 Prefill Worker
- 1 DP=16 Decode Worker

**Deployment Commands**:
```bash
export NAMESPACE=llm-d-wide-ep
kubectl create namespace ${NAMESPACE}
cd ${LLMD_PATH}/guides/wide-ep-lws
# For GKE with H200:
kubectl apply -k ./manifests/modelserver/gke -n ${NAMESPACE}
helm install llm-d-infpool -n ${NAMESPACE} -f ./manifests/inferencepool.values.yaml --set 'provider.name=gke' oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool --version v1.3.0
```

### 4. Tiered Prefix Cache

**Location**: `guides/tiered-prefix-cache/README.md`

**Use Cases**:
- Long context workloads
- High concurrency workloads
- Scenarios requiring extended cache capacity
- Multi-replica deployments needing shared cache

**Storage Tiers**:

#### CPU RAM Offloading
- **Guide**: `guides/tiered-prefix-cache/cpu/README.md`
- **Benefits**: Little operational overhead, more capacity than HBM, faster than recomputation
- **Recommended**: Always enable as first tier beyond HBM

#### Local Disk
- **Guide**: `guides/tiered-prefix-cache/storage/README.md`
- **Benefits**: Significantly increased cache capacity
- **Considerations**: Slower than CPU RAM, suitable for latency-tolerant workloads

#### Shared Storage
- **Guide**: `guides/tiered-prefix-cache/storage/README.md`
- **Benefits**: Extended capacity, shared KV-cache across nodes, persistence across restarts
- **Supported**: CephFS, GCP Lustre, IBM Storage Scale
- **Considerations**: Additional operational overhead, latency depends on storage system

**Recommendations**:
- Always set up HBM and CPU RAM tiers
- Consider third/fourth tier when cache needs exceed HBM + CPU RAM
- Order tiers by cache read/write latencies
- Keep frequently accessed caches close to accelerator

## Prerequisites

Before deploying any Well-lit Path, ensure these prerequisites are met:

### 0. Set LLMD_PATH Environment Variable

**CRITICAL**: You must set the `LLMD_PATH` environment variable to point to your llm-d repository clone location.

```bash
# Check if LLMD_PATH is set
echo $LLMD_PATH

# If not set, set it to your llm-d repository path
export LLMD_PATH=/path/to/your/llm-d

# Example:
# export LLMD_PATH=/Users/username/dev/llm-d
# export LLMD_PATH=/home/user/projects/llm-d

# Add to your shell profile for persistence
echo 'export LLMD_PATH=/path/to/your/llm-d' >> ~/.bashrc  # or ~/.zshrc
```

**If LLMD_PATH is not set**, the assistant will ask you to provide the full path to your llm-d repository clone.

### 1. Client Setup
**Guide**: `guides/prereq/client-setup/README.md`

Required tools:
- kubectl
- helm
- helmfile
- git

Configuration:
- HuggingFace token setup (create `llm-d-hf-token` secret with `HF_TOKEN` key)
- llm-d version selection

### 2. Infrastructure
**Guide**: `guides/prereq/infrastructure/README.md`

Requirements:
- Sufficient cluster resources for high-scale inference
- Optional: LeaderWorkerSet controller for multi-host inference
- Optional: High-speed inter-accelerator networking for P/D and Wide-EP

### 3. Gateway Provider
**Guide**: `guides/prereq/gateway-provider/README.md`

Supported providers:
- Istio (default)
- K-Gateway
- Agent Gateway

Configuration path: `guides/prereq/gateway-provider/common-configurations/`

### 4. Monitoring
**Guide**: `docs/monitoring/README.md`

Components:
- Prometheus
- Grafana
- Custom metrics collection

## Deployment Workflow

When a user requests llm-d deployment, follow this workflow. **IMPORTANT**: Always ask the user for configuration values before proceeding. If they don't know, suggest defaults.

### Step 1: Understand Requirements
Ask about:
- Model size and type (e.g., Llama-70B, DeepSeek-R1, Qwen3-32B)
- Workload characteristics (prompt length, concurrency, throughput needs)
- Hardware availability (GPU type, count, networking capabilities)
- Performance goals (latency, throughput, cost optimization)

### Step 2: Recommend Well-lit Path
Based on their use case:
- Default/general use → Intelligent Inference Scheduling
- Large models + long prompts → P/D Disaggregation
- Very large MoE models → Wide Expert Parallelism
- Long context/high concurrency → Add Tiered Prefix Cache

### Step 3: Gather Configuration Values
**ALWAYS ask the user for these environment variables**:

1. **NAMESPACE**: The Kubernetes namespace for deployment
   - Default suggestion: `llm-d` (short names recommended to avoid hostname length issues)
   - Example alternatives: `llm-d-inference-scheduler`, `llm-d-pd`, `llm-d-wide-ep`
   - Ask: "What namespace would you like to use for the deployment?"

2. **RELEASE_NAME_POSTFIX** (optional): Custom postfix for release names to support concurrent installs
   - Default: Empty (no postfix)
   - Example: `inference-scheduling-2`, `my-custom`
   - Ask: "Do you need a custom release name postfix for concurrent installations?"

3. **Hardware Backend** (if applicable): The type of accelerator to use
   - Options: `cuda` (default), `amd`, `xpu`, `hpu`, `tpu`, `cpu`
   - Ask: "What hardware accelerators are you using? (NVIDIA/AMD/Intel XPU/Intel Gaudi/TPU/CPU)"

4. **Gateway Provider**: The gateway control plane to use
   - Options: `istio` (default), `kgateway`, `agentgateway`, `gke`, `digitalocean`
   - Default suggestion: `istio`
   - Ask: "Which gateway provider would you like to use?"

5. **Storage Class** (for Tiered Cache deployments): The Kubernetes storage class for PVCs
   - Options: `default`, `lustre`, or custom storage class name
   - Default suggestion: `default`
   - Ask: "What storage class should be used for the PVC?" (only if deploying Tiered Cache)

If the user doesn't know, provide suggested defaults.

### Step 4: Ask About Configuration Customization

**IMPORTANT**: Before deployment, ask if the user wants to customize configuration:

**Ask**: "Would you like to customize any configuration parameters, or use the defaults?"

**Common customizations include**:

#### For Benchmark Workloads (config.yaml):
- **Model name**: Which model to benchmark (default: Qwen/Qwen3-32B)
- **Workload type**: Type of synthetic data (random, shared_prefix, etc.)
- **Load stages**: Request rates and durations
- **Input/Output distribution**: Prompt and response length parameters
  - `input_distribution`: min, max, mean, std, total_count
  - `output_distribution`: min, max, mean, std, total_count
- **API settings**: Streaming mode, completion type
- **Parallelism**: Number of parallel workload launcher pods

**Example config.yaml parameters** (from `guides/inference-scheduling/config.yaml`):
```yaml
endpoint:
  model: Qwen/Qwen3-32B           # Change to your model
  namespace: llm-d                # Your namespace
  base_url: http://...            # Gateway service URL

workload:
  sanity_random:
    load:
      stages:
        - rate: 1                 # Requests per second
          duration: 30            # Duration in seconds
    data:
      input_distribution:
        min: 10                   # Min prompt length
        max: 100                  # Max prompt length
        mean: 50                  # Mean prompt length
      output_distribution:
        min: 10                   # Min output length
        max: 100                  # Max output length
```

#### For Model Server Deployments (values.yaml):
- **Replica count**: Number of model server replicas
- **Resource requests/limits**: CPU, memory, GPU allocation
- **Model configuration**: Model name, tensor parallelism, pipeline parallelism
- **vLLM arguments**: Custom vLLM engine arguments
- **Node selectors/tolerations**: Pod placement constraints

**If user wants to customize**:
1. Read the relevant configuration file (config.yaml or values.yaml)
2. Read the Well-lit Path README.md for configuration guidance
3. Show the user the current values
4. Ask which parameters they want to change
5. Help them make the changes
6. Explain the impact of each change

**If user wants defaults**:
- Proceed with standard deployment using default configurations

### Step 5: Read the Well-lit Path Guide
**CRITICAL**: Always read the complete README.md from `guides/{well-lit-path-slug}/README.md` for:
- Detailed hardware requirements
- Exact deployment commands
- Special configuration needs
- Validation steps
- Troubleshooting guidance

**When to read the README**:
- Before providing deployment instructions
- When the user asks for more details
- When troubleshooting issues
- When configuring advanced options
- When the user needs to modify configuration files

### Step 6: Prepare Environment

```bash
# Set environment variables based on user input
export NAMESPACE=<user-provided-or-default>
export RELEASE_NAME_POSTFIX=<user-provided-or-empty>
```

**Always check if the namespace already exists before creating it.** Prefer to use an existing namespace.

```bash
kubectl get namespace ${NAMESPACE}
```

- If the namespace **already exists** → use it as-is. Do NOT recreate it. Proceed to Step 7.
- If the namespace **does NOT exist** → inform the user before creating it:

  > "The namespace `${NAMESPACE}` does not exist. I will create it now."

  Then create it:
  ```bash
  kubectl create namespace ${NAMESPACE}
  ```

### Step 7: Configure Infrastructure
- Ensure cluster has sufficient resources
- Install optional controllers (LeaderWorkerSet if needed for Wide-EP)
- Configure networking (RDMA if needed for P/D or Wide-EP)
- **Read the specific Well-lit Path README.md for detailed infrastructure requirements**

### Step 8: Deploy Gateway Provider
```bash
# Choose and deploy gateway provider
cd ${LLMD_PATH}/guides/prereq/gateway-provider
# Follow gateway-specific deployment instructions from README.md
```

### Step 9: Set Up Monitoring
```bash
# Install monitoring stack
# Configure Prometheus and Grafana
# See docs/monitoring/README.md for details
```

### Step 10: Create PVC (Always Required)

**A PVC is ALWAYS required for every deployment path.** This is mandatory regardless of whether you are deploying Tiered Prefix Cache, benchmarks, or a standard inference deployment.

**Step 10a: Check for existing PVC**

Always run this first:
```bash
kubectl get pvc -n ${NAMESPACE}
```

- If a PVC with the required name already exists and its STATUS is `Bound` → **skip to Step 11**.
- If no PVC exists, or the required PVC is not present / not `Bound` → **proceed to Step 10b to create it**.

**Step 10b: Gather PVC configuration from the user**

Ask the user for the following values (suggest defaults if they don't know):
- **PVC Name**: *(default: `llm-d-kv-cache-storage`)*
- **Storage Size**: *(default: `18000Gi` for cache, `100Gi` for benchmarks)*
- **Storage Class**: Use the value confirmed in Step 3, or ask: "What storage class should be used?" *(default: `default`)*

**Step 10c: Create and apply the PVC**

1. **Create the PVC YAML file** at `${LLMD_PATH}/pvc.yaml`:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: <user-provided-pvc-name>
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: <user-provided-size>
     storageClassName: <user-provided-storage-class>
   ```

2. **Apply the PVC**:
   ```bash
   cd ${LLMD_PATH}
   kubectl apply -f pvc.yaml -n ${NAMESPACE}
   ```

3. **Verify PVC is bound** before proceeding:
   ```bash
   kubectl get pvc -n ${NAMESPACE}
   # STATUS must show "Bound" before continuing to Step 11
   ```

   If STATUS remains `Pending`, diagnose with:
   ```bash
   kubectl describe pvc <pvc-name> -n ${NAMESPACE}
   ```
   Common causes: storage class does not exist, no available PersistentVolumes, or provisioner not running.

**Reference**: See [`guides/tiered-prefix-cache/storage/manifests/pvc.yaml`](${LLMD_PATH}/guides/tiered-prefix-cache/storage/manifests/pvc.yaml) for the template. For more details on storage backends and configuration, read `guides/tiered-prefix-cache/storage/README.md`.

### Step 11: Deploy Well-lit Path
```bash
# Navigate to chosen guide directory
cd ${LLMD_PATH}/guides/<well-lit-path-slug>

# Deploy using helmfile with appropriate flags
# Examples based on user's configuration:

# Default (CUDA GPUs with Istio):
helmfile apply -n ${NAMESPACE}

# With hardware backend:
helmfile apply -e <hardware-backend> -n ${NAMESPACE}
# Examples: -e amd, -e xpu, -e hpu, -e cpu, -e gke_tpu

# With gateway provider:
helmfile apply -e <gateway-provider> -n ${NAMESPACE}
# Examples: -e kgateway, -e agentgateway, -e digitalocean

# With custom release name postfix:
RELEASE_NAME_POSTFIX=<postfix> helmfile apply -n ${NAMESPACE}

# Combined example (AMD GPUs with K-Gateway):
helmfile apply -e amd -e kgateway -n ${NAMESPACE}
```

**IMPORTANT**: Read the specific Well-lit Path README.md for exact deployment commands and any special considerations.

### Step 12: Create HTTPRoute

**IMPORTANT**: HTTPRoute must be created AFTER deploying the Well-lit Path to ensure the Gateway and InferencePool resources exist.

1. **Determine which HTTPRoute file to use** based on gateway provider:
   - For `istio`, `kgateway`, or `digitalocean`: Use `httproute.yaml`
   - For `gke`: Use `httproute.gke.yaml`

2. **If using RELEASE_NAME_POSTFIX**, update the HTTPRoute file first**:
   
   The HTTPRoute references Gateway and InferencePool names that include the release name. You must update these references to match your custom names.
   
   ```bash
   # Example for inference-scheduling with custom postfix:
   sed -e "s/infra-inference-scheduling-inference-gateway/infra-${RELEASE_NAME_POSTFIX}-inference-gateway/g" \
       -e "s/gaie-inference-scheduling/gaie-${RELEASE_NAME_POSTFIX}/g" \
       httproute.yaml > httproute-custom.yaml
   
   kubectl apply -f httproute-custom.yaml -n ${NAMESPACE}
   ```

3. **If NOT using RELEASE_NAME_POSTFIX**, apply directly**:
   ```bash
   # For istio/kgateway/digitalocean:
   kubectl apply -f httproute.yaml -n ${NAMESPACE}
   
   # For GKE:
   kubectl apply -f httproute.gke.yaml -n ${NAMESPACE}
   ```

4. **Verify HTTPRoute is created**:
   ```bash
   kubectl get httproute -n ${NAMESPACE}
   ```

**HTTPRoute Template Structure** (example from `guides/inference-scheduling/httproute.yaml`):
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llm-d-inference-scheduling
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: infra-inference-scheduling-inference-gateway
  rules:
    - backendRefs:
        - group: inference.networking.k8s.io
          kind: InferencePool
          name: gaie-inference-scheduling
          port: 8000
          weight: 1
      timeouts:
        backendRequest: 0s
        request: 0s
      matches:
        - path:
            type: PathPrefix
            value: /
```

**Note**: Each Well-lit Path guide directory contains its own `httproute.yaml` and `httproute.gke.yaml` files with the correct resource names for that deployment. Always use the HTTPRoute file from the guide directory you're deploying.

### Step 13: Execute the Deployment Plan

**CRITICAL**: After the deployment plan has been created and confirmed by the user, execute ALL commands sequentially. Do NOT just show commands — run them one by one using the available tools, observe the output, and fix any errors before proceeding to the next step.

#### Execution Rules

1. **Run every command** from Steps 6–12 in order using `execute_command`.
2. **After each command**, inspect the output:
   - If the command **succeeds** → proceed to the next command.
   - If the command **fails** → diagnose the error, apply a fix, and re-run before continuing.
3. **Never skip a step** — each step depends on the previous one succeeding.
4. **Always use the LLMD_PATH environment variable** `${LLMD_PATH}` when navigating directories.

#### Common Errors and Fixes During Execution

| Error | Cause | Fix |
|-------|-------|-----|
| `namespace already exists` | Namespace was previously created | Safe to ignore; continue |
| `PVC already exists` | PVC was previously created | Run `kubectl get pvc -n ${NAMESPACE}` to confirm it is `Bound`; skip creation |
| `Error: INSTALLATION FAILED: cannot re-use a name that is still in use` | Helm release already installed | Run `helm upgrade` instead of `helm install`, or `helmfile apply` (idempotent) |
| `error: the server doesn't have a resource type "inferencepool"` | Gateway API CRDs not installed | Install the Gateway API CRDs before deploying |
| `Pending` pods / `0/N nodes available` | Insufficient cluster resources or missing node selectors | Check node labels, tolerations, and available GPU capacity |
| `CrashLoopBackOff` | Container startup failure | Run `kubectl logs <pod> -n ${NAMESPACE}` and `kubectl describe pod <pod> -n ${NAMESPACE}` to diagnose |
| `ImagePullBackOff` | Image not accessible | Verify image registry credentials and image name/tag |
| `helmfile: command not found` | helmfile not installed | Install helmfile: `brew install helmfile` (macOS) or follow https://helmfile.readthedocs.io |
| `Error from server (NotFound): httproute` | HTTPRoute CRD missing | Ensure Gateway API is installed in the cluster |

#### Execution Sequence

Run the following in order, checking output after each:

```bash
# 1. Set environment variables
export NAMESPACE=<confirmed-namespace>
export RELEASE_NAME_POSTFIX=<confirmed-postfix-or-empty>

# 2. Check if namespace exists; create only if it does not
kubectl get namespace ${NAMESPACE}
# If the above returns "Error from server (NotFound)", inform the user and then run:
# "The namespace ${NAMESPACE} does not exist. Creating it now."
kubectl create namespace ${NAMESPACE}
# If the namespace already exists, skip creation and proceed.

# 3. Check for existing PVC
kubectl get pvc -n ${NAMESPACE}
# If PVC is missing or not Bound, create it:
cd ${LLMD_PATH}
kubectl apply -f pvc.yaml -n ${NAMESPACE}
kubectl get pvc -n ${NAMESPACE}

# 4. Deploy gateway provider
cd ${LLMD_PATH}/guides/prereq/gateway-provider
# (follow gateway-specific commands from README.md)

# 5. Deploy the Well-lit Path
cd ${LLMD_PATH}/guides/<well-lit-path-slug>
helmfile apply -n ${NAMESPACE}
# Add -e flags as needed: -e <hardware-backend> -e <gateway-provider>

# 6. Apply HTTPRoute
cd ${LLMD_PATH}/guides/<well-lit-path-slug>
kubectl apply -f httproute.yaml -n ${NAMESPACE}
# For GKE: kubectl apply -f httproute.gke.yaml -n ${NAMESPACE}

# 7. Validate all resources
kubectl get pods -n ${NAMESPACE}
helm list -n ${NAMESPACE}
kubectl get httproute -n ${NAMESPACE}
kubectl get gateway -n ${NAMESPACE}
kubectl get inferencepool -n ${NAMESPACE}
```

#### Post-Execution Checks

After all commands complete successfully, verify the deployment is healthy:

```bash
# All pods should be Running or Completed
kubectl get pods -n ${NAMESPACE}

# All helm releases should show STATUS: deployed
helm list -n ${NAMESPACE}

# HTTPRoute should be Accepted and Programmed
kubectl get httproute -n ${NAMESPACE} -o wide

# Gateway should be Ready
kubectl get gateway -n ${NAMESPACE}
```

If any resource is not in the expected state, diagnose using:
```bash
kubectl describe <resource-type> <resource-name> -n ${NAMESPACE}
kubectl logs <pod-name> -n ${NAMESPACE} --previous
kubectl events -n ${NAMESPACE} --sort-by='.lastTimestamp'
```

### Step 14: Validate and Test
- Check pod status: `kubectl get pods -n ${NAMESPACE}`
- List helm releases: `helm list -n ${NAMESPACE}`
- Verify HTTPRoute: `kubectl get httproute -n ${NAMESPACE}`
- Check gateway: `kubectl get gateway -n ${NAMESPACE}`
- Verify InferencePool: `kubectl get inferencepool -n ${NAMESPACE}`
- Test inference endpoint (see `docs/getting-started-inferencing.md`)
- Run benchmark tests (if applicable)
- Monitor metrics

### Step 15: Provide Next Steps
- How to make inference requests (`docs/getting-started-inferencing.md`)
- How to run benchmarks (if applicable)
- How to monitor the deployment
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