---
name: llm-d-kubernetes-deployment
description: Deploy and troubleshoot llm-d (high-performance LLM inference) on Kubernetes or OpenShift clusters. Use this skill when users want to set up LLM serving infrastructure, deploy models for production inference, optimize LLM performance, fix deployment issues or pending pods, choose between deployment guides (inference scheduling, prefill/decode disaggregation, prefix caching), configure hardware-specific deployments (NVIDIA, AMD, Intel, TPU), or troubleshoot existing deployments. Handles both new deployments and fixing failed ones.
---

# llm-d Kubernetes/OpenShift Deployment Skill

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
> **Never silently create resources.** If you are unsure whether a resource already exists, check first, then notify before acting.

## What Not To Do

Critical rules to follow when deploying and managing llm-d:

1. **Do NOT change cluster-level definitions** 
All changes must be made exclusively inside the designated project namespace. Never modify cluster-wide resources (e.g., ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes, or any resource outside the target namespace). Scope every `kubectl apply`, `helm install`, and `helmfile apply` command to the target namespace using `-n ${NAMESPACE}`.

2. **Do NOT modify any existing code you did not create** 
 Only create new files and modify them as needed. Never edit pre-existing files in the repository (e.g., existing `values.yaml`, `helmfile.yaml`, `httproute.yaml`, `README.md`, or any other committed file). If customization is required, create a new file (e.g., `values-custom.yaml`, `httproute-custom.yaml`) and reference it instead.

## Overview

This skill enables AI agents to guide users through deploying llm-d on Kubernetes or OpenShift clusters using Well-lit Path guides. llm-d provides tested and benchmarked recipes for serving large language models at peak performance with production best practices.

llm-d provides Well-lit Path deployment guides located in the `guides/` directory. Each guide has a README.md file with the prefix "Well-lit Path:" in its title, containing tested and benchmarked recipes for specific deployment strategies.

## When to Use This Skill

Activate this skill when users need to:
- Deploy LLM inference on Kubernetes or OpenShift
- Optimize LLM serving performance
- Set up production-ready model serving infrastructure
- Choose between different deployment strategies
- Configure hardware-specific deployments (NVIDIA, AMD, Intel, TPU, CPU)
- Troubleshoot failed deployments or pending pods
- Fix deployment issues or errors
- Update or modify existing deployments

## Prerequisites

Each Well-lit Path guide contains its own prerequisites section. Common prerequisites include:

1. **LLMD_PATH Environment Variable** - Point to your llm-d repository clone
2. **Client Tools** - kubectl/oc, helm, helmfile, git (see `guides/prereq/client-setup/README.md`)
3. **HuggingFace Token** - Secret `llm-d-hf-token` with key `HF_TOKEN` in target namespace
4. **Gateway Provider** - Istio, K-Gateway, or Agent Gateway (see `guides/prereq/gateway-provider/README.md`)
5. **Infrastructure** - Sufficient cluster resources (see `guides/prereq/infrastructure/README.md`)
6. **Monitoring** (optional) - Prometheus and Grafana (see `docs/monitoring/README.md`)

**Refer to the specific guide's README.md for detailed prerequisites.**
## Platform Detection

Before starting any deployment, detect the platform type and verify access:

```bash
# Check if OpenShift
if oc version 2>/dev/null; then
  echo "Platform: OpenShift"
  PLATFORM="openshift"
  CLI_CMD="oc"
else
  echo "Platform: Kubernetes"
  PLATFORM="kubernetes"
  CLI_CMD="kubectl"
fi

# If OpenShift, verify login
if [ "$PLATFORM" = "openshift" ]; then
  oc whoami 2>/dev/null || {
    echo "ERROR: Not logged in to OpenShift"
    echo "Please run: oc login --token=<token> --server=<server>"
    exit 1
  }
fi
```

**Platform-specific adaptations:**
- **OpenShift**: Use `oc` commands, check `oc project`, consider Routes alongside HTTPRoutes
- **Kubernetes**: Use `kubectl` commands, check context, use Ingress/HTTPRoute

## Deployment Workflow

When a user requests llm-d deployment, follow this streamlined workflow:

### 1. Setup and Discovery

#### 1.1 Verify llm-d Repository Access
   1. Check `LLMD_PATH` environment variable
   2. If not set, check if current directory is llm-d repository
   3. If not found, ask user for repository path
   4. If user doesn't have a clone, suggest cloning from https://github.com/llm-d/llm-d and ask for clone location

   Set LLMD_PATH to the user path.

#### 1.2 Discover Available Guides

```bash
# Find all Well-lit Path guides in one command
find ${LLMD_PATH}/guides -maxdepth 2 -name "README.md" \
  -not -path "*/prereq/*" -not -path "*/recipes/*" \
  -exec sh -c '
    if head -n 5 "$1" | grep -q "^# Well-lit Path:"; then
      guide_dir=$(dirname "$1")
      guide_name=$(basename "$guide_dir")
      title=$(head -n 5 "$1" | grep "^# Well-lit Path:" | sed "s/^# Well-lit Path: //")
      echo "$guide_name|$title"
    fi
  ' _ {} \;
```

**Present discovered guides to user with descriptions.**

**Default suggestion**: If `inference-scheduling` guide exists, suggest it as the recommended starting point for production deployments.

### 2. Configuration Detection

#### 2.1 Detect or Set Namespace

```bash
# For OpenShift - check current project
if [ "$PLATFORM" = "openshift" ]; then
  NAMESPACE=$(oc project -q 2>/dev/null)
fi

# For Kubernetes - check current context namespace
if [ -z "$NAMESPACE" ]; then
  NAMESPACE=$(kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null)
fi

# If still not set, use default or ask user
if [ -z "$NAMESPACE" ]; then
  echo "No namespace detected. Please specify namespace."
  # Ask user, then: export NAMESPACE=user-provided-namespace
fi

echo "Using namespace: $NAMESPACE"
```

#### 2.2 Auto-Detect Hardware

```bash
# Detect accelerator type
HARDWARE=$(kubectl get nodes -o json | jq -r '
  .items[].status.capacity | 
  if ."nvidia.com/gpu" then "nvidia"
  elif ."amd.com/gpu" then "amd"  
  elif ."gpu.intel.com/i915" then "xpu"
  elif ."habana.ai/gaudi" then "hpu"
  elif ."google.com/tpu" then "tpu"
  else "cpu"
  end' | head -n1)

echo "Detected hardware: $HARDWARE"

# Get hardware count
case $HARDWARE in
  nvidia)
    GPU_COUNT=$(kubectl get nodes -o json | jq '[.items[].status.capacity."nvidia.com/gpu" // "0" | tonumber] | add')
    echo "NVIDIA GPUs available: $GPU_COUNT"
    ;;
  amd)
    GPU_COUNT=$(kubectl get nodes -o json | jq '[.items[].status.capacity."amd.com/gpu" // "0" | tonumber] | add')
    echo "AMD GPUs available: $GPU_COUNT"
    ;;
  tpu)
    TPU_COUNT=$(kubectl get nodes -o json | jq '[.items[].status.capacity."google.com/tpu" // "0" | tonumber] | add')
    echo "TPUs available: $TPU_COUNT"
    ;;
esac
```

#### 2.3 Auto-Detect Gateway Provider

```bash
# Check for gateway providers in priority order
if kubectl get namespace istio-system 2>/dev/null >/dev/null; then
  GATEWAY="istio"
  echo "Detected gateway: Istio"
elif kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null >/dev/null && \
     kubectl get pods -n kube-system -l app=gateway 2>/dev/null | grep -q Running; then
  GATEWAY="kgateway"
  echo "Detected gateway: K-Gateway"
elif kubectl get deployment -n gateway-system agent-gateway 2>/dev/null >/dev/null; then
  GATEWAY="agentgateway"
  echo "Detected gateway: Agent Gateway"
elif kubectl get nodes -o json | jq -r '.items[0].metadata.labels["cloud.google.com/gke-nodepool"]' 2>/dev/null | grep -q .; then
  GATEWAY="gke"
  echo "Detected gateway: GKE Gateway"
else
  GATEWAY="none"
  echo "WARNING: No gateway provider detected"
fi
```

#### 2.4 Auto-Detect Storage Class

```bash
# Get default storage class
STORAGE_CLASS=$(kubectl get storageclass -o json | \
  jq -r '.items[] | select(.metadata.annotations["storageclass.kubernetes.io/is-default-class"]=="true") | .metadata.name' | \
  head -n1)

# If no default, try GKE-specific detection
if [ -z "$STORAGE_CLASS" ] && [ "$GATEWAY" = "gke" ]; then
  STORAGE_CLASS=$(kubectl get storageclass -o json | \
    jq -r '.items[] | select(.metadata.name | contains("standard")) | .metadata.name' | \
    head -n1)
fi

# If still not found, list available
if [ -z "$STORAGE_CLASS" ]; then
  echo "No default storage class found. Available storage classes:"
  kubectl get storageclass -o name
else
  echo "Detected storage class: $STORAGE_CLASS"
fi
```

**Present detected configuration to user:**

```
Detected configuration:
- Platform: {openshift/kubernetes}
- Namespace: {detected-namespace}
- Hardware: {detected-hardware} ({count} accelerators)
- Gateway Provider: {detected-gateway}
- Storage Class: {detected-storage-class}
```

**Only ask user for input if:**
1. Auto-detection failed for any component
2. User wants to override detected values
3. Multiple valid options exist and choice is ambiguous

### 3. Prerequisites Verification

#### 3.1 Verify Namespace

```bash
# Check if namespace exists
if ! $CLI_CMD get namespace ${NAMESPACE} 2>/dev/null >/dev/null; then
  echo "Namespace ${NAMESPACE} does not exist. Creating it..."
  $CLI_CMD create namespace ${NAMESPACE}
else
  echo "Namespace ${NAMESPACE} exists"
fi
```

#### 3.2 Verify HuggingFace Token Secret

```bash
# Check if secret exists
if $CLI_CMD get secret llm-d-hf-token -n ${NAMESPACE} 2>/dev/null >/dev/null; then
  echo "HuggingFace token secret exists"
  
  # Verify it has the correct key
  if $CLI_CMD get secret llm-d-hf-token -n ${NAMESPACE} -o jsonpath='{.data.HF_TOKEN}' 2>/dev/null | base64 -d | grep -q .; then
    echo "HuggingFace token secret is valid"
  else
    echo "WARNING: HuggingFace token secret exists but HF_TOKEN key is missing or empty"
  fi
else
  echo "ERROR: HuggingFace token secret not found"
  echo "Please create it with:"
  echo "$CLI_CMD create secret generic llm-d-hf-token --from-literal=HF_TOKEN=<your-token> -n ${NAMESPACE}"
  exit 1
fi
```

#### 3.3 Verify Storage (if required by guide)

```bash
# Check if guide requires PVC
if [ -f "${LLMD_PATH}/guides/${GUIDE_NAME}/manifests/pvc.yaml" ]; then
  echo "Guide requires persistent storage"
  
  # Check for existing PVC
  if $CLI_CMD get pvc -n ${NAMESPACE} 2>/dev/null | grep -q Bound; then
    PVC_NAME=$($CLI_CMD get pvc -n ${NAMESPACE} -o jsonpath='{.items[0].metadata.name}')
    echo "Found existing PVC: $PVC_NAME (Bound)"
  else
    echo "No bound PVC found. Creating PVC..."
    
    # Apply PVC manifest (may need to customize storage class)
    $CLI_CMD apply -f ${LLMD_PATH}/guides/${GUIDE_NAME}/manifests/pvc.yaml -n ${NAMESPACE}
    
    # Wait for PVC to be Bound
    echo "Waiting for PVC to be bound (timeout: 5 minutes)..."
    $CLI_CMD wait --for=condition=Bound pvc/$(kubectl get pvc -n ${NAMESPACE} -o jsonpath='{.items[0].metadata.name}') \
      -n ${NAMESPACE} --timeout=300s
  fi
fi
```

#### 3.4 Verify Gateway Provider

```bash
# Verify gateway provider is ready
case $GATEWAY in
  istio)
    if ! kubectl get pods -n istio-system -l app=istiod | grep -q Running; then
      echo "ERROR: Istio control plane not running"
      exit 1
    fi
    ;;
  kgateway)
    if ! kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null >/dev/null; then
      echo "ERROR: Gateway API CRDs not installed"
      exit 1
    fi
    ;;
  gke)
    # GKE gateway is managed, just verify we're on GKE
    if ! kubectl get nodes -o json | jq -r '.items[0].metadata.labels["cloud.google.com/gke-nodepool"]' | grep -q .; then
      echo "ERROR: Not running on GKE but GKE gateway was detected"
      exit 1
    fi
    ;;
esac

echo "Gateway provider verified: $GATEWAY"
```

### 4. Deployment Execution

#### 4.1 Navigate to Guide Directory

```bash
cd ${LLMD_PATH}/guides/${GUIDE_NAME}
echo "Working directory: $(pwd)"
```

#### 4.2 Deploy with Helmfile

```bash
# Determine environment flags based on detected configuration
ENV_FLAGS=""

# Add hardware environment if not default
case $HARDWARE in
  nvidia) ENV_FLAGS="$ENV_FLAGS -e cuda" ;;
  amd) ENV_FLAGS="$ENV_FLAGS -e rocm" ;;
  xpu) ENV_FLAGS="$ENV_FLAGS -e xpu" ;;
  hpu) ENV_FLAGS="$ENV_FLAGS -e hpu" ;;
  tpu) ENV_FLAGS="$ENV_FLAGS -e tpu" ;;
esac

# Add gateway environment if not default
case $GATEWAY in
  istio) ENV_FLAGS="$ENV_FLAGS -e istio" ;;
  kgateway) ENV_FLAGS="$ENV_FLAGS -e kgateway" ;;
  agentgateway) ENV_FLAGS="$ENV_FLAGS -e agentgateway" ;;
esac

# Deploy with helmfile
echo "Deploying with helmfile..."
echo "Command: helmfile apply -n ${NAMESPACE}${ENV_FLAGS}"
helmfile apply -n ${NAMESPACE}${ENV_FLAGS}
```

#### 4.3 Apply HTTPRoute

```bash
# Determine which HTTPRoute to apply
if [ "$GATEWAY" = "gke" ] && [ -f "httproute.gke.yaml" ]; then
  HTTPROUTE_FILE="httproute.gke.yaml"
elif [ -f "httproute.yaml" ]; then
  HTTPROUTE_FILE="httproute.yaml"
else
  echo "WARNING: No HTTPRoute file found"
  HTTPROUTE_FILE=""
fi

# Apply HTTPRoute if found
if [ -n "$HTTPROUTE_FILE" ]; then
  echo "Applying HTTPRoute from $HTTPROUTE_FILE..."
  $CLI_CMD apply -f $HTTPROUTE_FILE -n ${NAMESPACE}
fi

# For OpenShift, also check for Route
if [ "$PLATFORM" = "openshift" ] && [ -f "route.yaml" ]; then
  echo "Applying OpenShift Route..."
  oc apply -f route.yaml -n ${NAMESPACE}
fi
```

### 5. Validation

#### 5.1 Check Pod Status

```bash
# Wait a moment for pods to start
sleep 5

# Get pod status
echo "Checking pod status..."
$CLI_CMD get pods -n ${NAMESPACE}

# Check for non-Running pods
NON_RUNNING=$($CLI_CMD get pods -n ${NAMESPACE} -o json | \
  jq -r '.items[] | select(.status.phase != "Running") | .metadata.name')

if [ -n "$NON_RUNNING" ]; then
  echo "WARNING: Some pods are not running:"
  echo "$NON_RUNNING"
  echo "Run troubleshooting workflow to diagnose issues"
fi
```

#### 5.2 Verify Resources

```bash
# Check all llm-d resources
echo "Verifying resources..."
$CLI_CMD get inferencepool,gateway,httproute,svc,pvc -n ${NAMESPACE}

# Verify InferencePool status
if $CLI_CMD get inferencepool -n ${NAMESPACE} 2>/dev/null >/dev/null; then
  POOL_STATUS=$($CLI_CMD get inferencepool -n ${NAMESPACE} -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}')
  if [ "$POOL_STATUS" = "True" ]; then
    echo "✓ InferencePool is Ready"
  else
    echo "✗ InferencePool is not Ready"
  fi
fi

# Verify Gateway status
if $CLI_CMD get gateway -n ${NAMESPACE} 2>/dev/null >/dev/null; then
  GW_STATUS=$($CLI_CMD get gateway -n ${NAMESPACE} -o jsonpath='{.items[0].status.conditions[?(@.type=="Programmed")].status}')
  if [ "$GW_STATUS" = "True" ]; then
    echo "✓ Gateway is Programmed"
  else
    echo "✗ Gateway is not Programmed"
  fi
fi

# Verify HTTPRoute status
if $CLI_CMD get httproute -n ${NAMESPACE} 2>/dev/null >/dev/null; then
  ROUTE_STATUS=$($CLI_CMD get httproute -n ${NAMESPACE} -o jsonpath='{.items[0].status.parents[0].conditions[?(@.type=="Accepted")].status}')
  if [ "$ROUTE_STATUS" = "True" ]; then
    echo "✓ HTTPRoute is Accepted"
  else
    echo "✗ HTTPRoute is not Accepted"
  fi
fi
```

#### 5.3 Test Connectivity

```bash
# Get gateway address
GATEWAY_ADDR=$($CLI_CMD get gateway -n ${NAMESPACE} -o jsonpath='{.items[0].status.addresses[0].value}' 2>/dev/null)

if [ -n "$GATEWAY_ADDR" ]; then
  echo "Gateway address: $GATEWAY_ADDR"
  echo "Testing endpoint..."
  
  # Test /v1/models endpoint
  if curl -s -f http://${GATEWAY_ADDR}/v1/models >/dev/null; then
    echo "✓ Inference endpoint is responding"
  else
    echo "✗ Inference endpoint is not responding"
    echo "This may be normal if pods are still starting up"
  fi
else
  echo "WARNING: Could not determine gateway address"
fi
```

**Success Criteria:**
- ✅ All pods in Running state with N/N ready
- ✅ InferencePool shows Ready status
- ✅ Gateway shows Programmed status
- ✅ HTTPRoute shows Accepted status
- ✅ Inference endpoint responds to requests


### 6 Generate Deployment Script

After successful validation, generate a reusable deployment script:

```bash
# Ask user for script location
SCRIPT_PATH="${LLMD_PATH}/deployments/deploy-${NAMESPACE}-$(date +%Y%m%d-%H%M%S).sh"
echo "Generating deployment script at: $SCRIPT_PATH"

# Create deployments directory if it doesn't exist
mkdir -p $(dirname $SCRIPT_PATH)

# Generate script
cat > $SCRIPT_PATH << 'EOF'
#!/bin/bash
# llm-d Deployment Script
# Generated: $(date)
# Guide: ${GUIDE_NAME}
# Namespace: ${NAMESPACE}
# Platform: ${PLATFORM}

set -e  # Exit on error

# Configuration
export LLMD_PATH="${LLMD_PATH}"
export NAMESPACE="${NAMESPACE}"
export PLATFORM="${PLATFORM}"
export CLI_CMD="${CLI_CMD}"
export HARDWARE="${HARDWARE}"
export GATEWAY="${GATEWAY}"

echo "========================================="
echo "llm-d Deployment Script"
echo "========================================="
echo "Guide: ${GUIDE_NAME}"
echo "Namespace: ${NAMESPACE}"
echo "Platform: ${PLATFORM}"
echo "Hardware: ${HARDWARE}"
echo "Gateway: ${GATEWAY}"
echo "========================================="

# [Include all executed commands from deployment]
# Step 1: Verify namespace
# Step 2: Verify prerequisites
# Step 3: Deploy with helmfile
# Step 4: Apply HTTPRoute
# Step 5: Validate deployment

echo "========================================="
echo "Deployment complete!"
echo "========================================="
EOF

chmod +x $SCRIPT_PATH
echo "Deployment script created: $SCRIPT_PATH"

```


## Common Issues and Fixes

| Issue | Diagnosis Command | Fix |
|-------|------------------|-----|
| **Pods Pending** | `$CLI_CMD describe pod <pod> -n ${NAMESPACE}` shows "Insufficient resources" | Check node capacity: `$CLI_CMD describe nodes \| grep -A 5 "Allocated resources"`. Adjust resource requests or add nodes. |
| **Pods Pending** | Events show "no nodes available" or "MatchNodeSelector" | Check node selectors: `$CLI_CMD get pod <pod> -n ${NAMESPACE} -o yaml \| grep -A 5 nodeSelector`. Verify nodes have required labels. |
| **CrashLoopBackOff** | Logs show "HF_TOKEN not found" | Create HuggingFace token secret: `$CLI_CMD create secret generic llm-d-hf-token --from-literal=HF_TOKEN=<token> -n ${NAMESPACE}` |
| **CrashLoopBackOff** | Logs show "CUDA error" or "GPU not found" | Verify GPU drivers and allocation. See Hardware-Specific Troubleshooting below. |
| **ImagePullBackOff** | Events show "unauthorized" or "not found" | Check image name and registry credentials: `$CLI_CMD describe pod <pod> -n ${NAMESPACE} \| grep Image` |
| **PVC Pending** | `$CLI_CMD describe pvc <pvc> -n ${NAMESPACE}` shows "no storage class" | Create or specify storage class. Check available: `$CLI_CMD get storageclass` |
| **Gateway not Programmed** | `$CLI_CMD describe gateway -n ${NAMESPACE}` shows errors | Check gateway provider is running. Verify Gateway API CRDs installed. |
| **HTTPRoute not Accepted** | `$CLI_CMD describe httproute -n ${NAMESPACE}` shows errors | Verify gateway reference is correct. Check backend services exist. |

## Additional Resources

- **Quickstart**: `guides/QUICKSTART.md`
- **Architecture**: https://llm-d.ai/docs/architecture
- **Gateway Customization**: `docs/customizing-your-gateway.md`
- **Inference Gateway**: https://github.com/kubernetes-sigs/gateway-api-inference-extension
- **Inference Scheduler Architecture**: https://github.com/llm-d/llm-d-inference-scheduler/blob/main/docs/architecture.md
- **Monitoring Setup**: `docs/monitoring/README.md`
- **Performance Tuning**: `docs/performance-tuning.md`

---

## Security Considerations

- Run deployments in isolated namespaces
- Use RBAC for access control
- Secure HuggingFace tokens as Kubernetes secrets
- Enable network policies for pod-to-pod communication
- Use TLS for gateway ingress
- Audit all deployments and changes
- Regularly update container images for security patches
- Use Pod Security Standards/Policies
- Limit resource requests to prevent resource exhaustion
- Monitor for suspicious activity

---

## OpenShift-Specific Notes

When deploying on OpenShift:

1. **Use `oc` command** - While `kubectl` works, `oc` provides OpenShift-specific features
2. **Projects vs Namespaces** - OpenShift uses "projects" which are namespaces with additional features
3. **Security Context Constraints** - OpenShift has stricter security by default. May need to adjust SCCs.
4. **Routes vs Ingress** - OpenShift Routes are preferred over Ingress/HTTPRoute for external access
5. **Image Streams** - Consider using OpenShift Image Streams for container images
6. **Service Mesh** - OpenShift Service Mesh (based on Istio) may already be available

**OpenShift-specific commands:**
```bash
# Login to cluster
oc login --token=<token> --server=<server>

# Get current project
oc project

# Create new project (creates namespace + additional resources)
oc new-project <name>

# Create route for external access
oc expose service <service-name>

# Check security context constraints
oc get scc
oc describe scc restricted
