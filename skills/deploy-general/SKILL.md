---
name: llm-d-general-deployment
description: Generate custom llm-d deployment configurations for Kubernetes with flexible model, hardware, and gateway settings. Creates helmfile.yaml, httproute.yaml, and deployment README based on user requirements.
---

# llm-d General Deployment Skill

## 📋 Command Execution Notice

**Before executing any command, I will:**
1. **Explain what the command does** - A clear description of the command's purpose and expected outcome
2. **Show the actual command** - The exact command that will be executed
3. **Explain why it's needed** - How this command fits into the overall deployment workflow

This ensures you understand each step before it happens and can verify the actions align with your intentions.

> ## 🔔 ALWAYS NOTIFY THE USER BEFORE CREATING ANYTHING
>
> **RULE**: Before creating ANY resource — including files, namespaces, PVCs, Helm releases, HTTPRoutes, or any Kubernetes object — you MUST first tell the user what you are about to create and why.
>
> **Format to use before every creation action**:
> > "I am about to create `<resource-type>` named `<name>` because `<reason>`. Proceeding now."
>
> **Never silently create resources.** If you are unsure whether a resource already exists, check first, then notify before acting.

This skill enables AI agents to generate custom llm-d deployment configurations for Kubernetes clusters with general configurations that are not part of the well-lit paths. The agent will create deployment files (helmfile.yaml, httproute.yaml) and a README documenting the configuration.


## What Not To Do
Critical rules to follow when deploying and managing llm-d:

1. **Do NOT change cluster-level definitions** — All changes must be made exclusively inside the designated project namespace. Never modify cluster-wide resources (e.g., ClusterRoles, ClusterRoleBindings, StorageClasses, Nodes, or any resource outside the target namespace). Scope every `kubectl apply`, `helm install`, and `helmfile apply` command to the target namespace using `-n ${NAMESPACE}`.

2. **Do NOT modify any existing code you did not create** — Only create new files and modify them as needed. Never edit pre-existing files in the repository (e.g., existing `values.yaml`, `helmfile.yaml`, `httproute.yaml`, `README.md`, or any other committed file). If customization is required, create a new file (e.g., `values-custom.yaml`, `httproute-custom.yaml`) and reference it instead.

## Overview

Unlike the well-lit path deployment skill which uses pre-tested guides, this skill generates custom deployment configurations based on user requirements. The agent will:

1. Gather deployment requirements from the user
2. Generate a helmfile.yaml with appropriate configurations
3. Generate httproute.yaml for traffic routing
4. Create a README documenting the deployment setup and usage

## When to Use This Skill

Activate this skill when users need to:
- Deploy llm-d with custom configurations not covered by well-lit paths
- Experiment with different model configurations
- Create deployment configurations for specific hardware setups
- Generate deployment files for custom use cases
- Deploy models with non-standard settings

## Prerequisites

Before using this skill, ensure:

1. **Client Tools** - kubectl, helm, helmfile installed
2. **Kubernetes Cluster** - Access to a Kubernetes cluster with sufficient resources
3. **HuggingFace Token** - Secret `llm-d-hf-token` with key `HF_TOKEN` in target namespace
4. **Gateway Provider** - Istio, K-Gateway, Agent Gateway, or GKE Gateway installed
5. **Gateway API CRDs** - Gateway API v1.4.0+ and Inference Extension v1.3.0+ CRDs installed
6. **Target Directory** - A directory where deployment files will be generated

**Reference Documentation:**
- [llm-d Project](https://github.com/llm-d/llm-d)
- [Client Setup Guide](https://github.com/llm-d/llm-d/blob/main/guides/prereq/client-setup/README.md)
- [Gateway Provider Setup](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/README.md)
- [Infrastructure Requirements](https://github.com/llm-d/llm-d/blob/main/guides/prereq/infrastructure/README.md)
- [Well-lit Paths Overview](https://github.com/llm-d/llm-d/blob/main/guides/README.md)

## Deployment Workflow

When a user requests a custom llm-d deployment, follow this workflow:

### Step 1: Gather Deployment Requirements

Ask the user for the following information. Use `ask_followup_question` if any required field is missing.

**VALIDATION RULE**: Before proceeding to Step 2, verify ALL required fields are provided. If any are missing, use `ask_followup_question` to gather them.

#### Required Information:
1. **Deployment Name** - A unique name for this deployment (e.g., "custom-llama-deployment")
   - Must be lowercase, alphanumeric with hyphens only
   - Max 20 characters
2. **Target Namespace** - Kubernetes namespace for deployment (default: "llmd-custom")
3. **Model Configuration**:
   - Model URI (HuggingFace model ID or path, e.g., "hf://Qwen/Qwen3-32B")
   - Model name (e.g., "Qwen/Qwen3-32B")
   - Model size (storage requirement, e.g., "80Gi")
4. **Hardware Configuration**:
   - Accelerator type: `nvidia` | `amd` | `intel-i915` | `intel-xe` | `habana` | `tpu` | `cpu`
   - Number of GPUs per replica (tensor parallelism, default: 2)
   - Number of replicas (data parallelism, default: 8)
5. **Gateway Provider**:
   - Gateway type: `istio` (default) | `kgateway` | `agentgateway` | `gke`

#### Optional Information (with defaults):
1. **Advanced vLLM Settings**:
   - GPU memory utilization: `0.95` (range: 0.0-1.0)
   - Additional vLLM arguments: `[]` (e.g., ["--max-model-len=4096"])
   - Custom container image: Auto-selected based on accelerator type
2. **Resource Limits**:
   - CPU limits/requests: `32` cores (adjust based on model size)
   - Memory limits/requests: `100Gi` (adjust based on model size)
3. **Prefill/Decode Configuration**:
   - Enable disaggregation: `false` (set true for PD separation)
   - Prefill replicas: `2` (if disaggregation enabled)
   - Decode replicas: `8` (if disaggregation enabled)
4. **Storage Configuration**:
   - Storage class: Cluster default
   - PVC size: Auto-calculated from model size
5. **Monitoring**:
   - Enable Prometheus monitoring: `true`

**Placeholder Value Reference Table**:
| Placeholder | Valid Values | Default |
|-------------|--------------|---------|
| `<monitoring-enabled>` | `true` \| `false` | `true` |
| `<accelerator-type>` | `nvidia` \| `amd` \| `intel-i915` \| `intel-xe` \| `habana` \| `tpu` \| `cpu` | `nvidia` |
| `<gpu-memory-utilization>` | `0.0` to `1.0` | `0.95` |
| `<tensor-parallelism>` | Integer ≥ 1 | `2` |
| `<replicas>` | Integer ≥ 1 | `8` |
| `<cpu-limit>` | String (e.g., `32`) | `32` |
| `<memory-limit>` | String (e.g., `100Gi`) | `100Gi` |
| `<shm-size>` | String (e.g., `20Gi`) | `20Gi` |

### Step 2: Generate helmfile.yaml

**IMPORTANT - Placeholder Replacement**: Before writing any files, replace ALL placeholders with actual values:
- `<namespace>` → User-provided namespace
- `<deployment-name>` → User-provided deployment name
- `<model-uri>` → User-provided model URI
- `<model-name>` → User-provided model name
- All other `<placeholders>` → Corresponding values from user input or defaults

Based on the gathered requirements, generate a `helmfile.yaml` file with Go template syntax.

**Note**: This helmfile uses Helm charts from the llm-d project repositories. No local clone is required.

**Helm Chart Repositories:**
- **llm-d-infra**: `https://llm-d-incubation.github.io/llm-d-infra/` - Infrastructure components (Gateway, monitoring)
- **llm-d-modelservice**: `https://llm-d-incubation.github.io/llm-d-modelservice/` - Model server deployment
- **inferencepool**: `oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool` - Inference scheduler

**Reference**: [llm-d Helm Charts](https://github.com/llm-d/llm-d/tree/main/guides/inference-scheduling)

```yaml
# Gateway provider configurations
# For Istio example, see: https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/istio.yaml
# For K-Gateway example, see: https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/kgateway.yaml
environments:
  istio:
    values:
      - provider:
          name: istio
          istio:
            destinationRule:
              enabled: true
              connectionPool:
                tcp:
                  maxConnections: 100000
                http:
                  http1MaxPendingRequests: 100000
                  http2MaxRequests: 100000
                  maxRequestsPerConnection: 0
              trafficPolicy:
                connectionPool:
                  tcp:
                    maxConnections: 100000
                  http:
                    http1MaxPendingRequests: 100000
                    http2MaxRequests: 100000
                    maxRequestsPerConnection: 0
      - gateway:
          service:
            type: LoadBalancer
          resources:
            requests:
              cpu: "2"
              memory: 2Gi
            limits:
              cpu: "4"
              memory: 4Gi
  kgateway:
    values:
      - provider:
          name: kgateway
      - gateway:
          service:
            type: LoadBalancer
  default:
    values:
      - provider:
          name: istio
      - gateway:
          service:
            type: LoadBalancer

---

{{- $ns := .Namespace | default "<namespace>" -}}
{{- $rn := (env "RELEASE_NAME_POSTFIX") | default "<deployment-name>" -}}

repositories:
  - name: llm-d-modelservice
    url: https://llm-d-incubation.github.io/llm-d-modelservice/
  - name: llm-d-infra
    url: https://llm-d-incubation.github.io/llm-d-infra/

releases:
  - name: {{ printf "infra-%s" $rn | quote }}
    namespace: {{ $ns }}
    chart: llm-d-infra/llm-d-infra
    version: v1.3.6
    installed: true
    labels:
      type: infrastructure
      kind: inference-stack
    values:
      - gateway:
          {{ .Environment.Values.gateway | toYaml | nindent 10 }}

  - name: {{ printf "gaie-%s" $rn | quote }}
    namespace: {{ $ns }}
    chart: oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool
    version: v1.3.0
    installed: true
    needs:
      - {{ printf "infra-%s" $rn | quote }}
    values:
      - <deployment-name>/gaie-values.yaml
    {{- if eq .Environment.Name "istio" }}
      - provider:
          name: {{ .Environment.Values.provider.name }}
          istio:
            {{ .Environment.Values.provider.istio | toYaml | nindent 12 }}
      - provider:
          istio:
            destinationRule:
              host: {{ printf "gaie-%s-epp.%s.svc.cluster.local" $rn $ns | quote }}
    {{- end }}
    labels:
      kind: inference-stack

  - name: {{ printf "ms-%s" $rn | quote }}
    namespace: {{ $ns }}
    chart: llm-d-modelservice/llm-d-modelservice
    version: v0.4.5
    installed: true
    needs:
      - {{ printf "infra-%s" $rn | quote }}
      - {{ printf "gaie-%s" $rn | quote }}
    values:
      - <deployment-name>/ms-values.yaml
    labels:
      kind: inference-stack
```

**Key Configuration Points**:
- Replace `<namespace>` with the target namespace
- Replace `<deployment-name>` with the deployment name
- Gateway configurations are embedded (no external file dependencies)
- Chart versions: infra v1.3.6, inferencepool v1.3.0, modelservice v0.4.5
- Use `-e istio` or `-e kgateway` flag with helmfile to select gateway provider

**Example helmfile commands:**
```bash
# Deploy with Istio (default)
helmfile apply -n ${NAMESPACE}

# Deploy with K-Gateway
helmfile apply -e kgateway -n ${NAMESPACE}
```

### Step 3: Generate Supporting Values Files

Create a directory structure for the deployment:
```
<deployment-directory>/
├── helmfile.yaml
├── httproute.yaml
├── README.md
└── <deployment-name>/
    ├── gaie-values.yaml
    └── ms-values.yaml
```

**Reference Examples:**
- [GAIE values example](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/gaie-inference-scheduling/values.yaml)
- [Model server values example](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values.yaml)

#### Generate gaie-values.yaml:

**CRITICAL**: The `matchLabels` in this file MUST exactly match the `labels` in ms-values.yaml. Mismatched labels will cause deployment failure.

```yaml
inferenceExtension:
  replicas: 1
  flags:
    kv-cache-usage-percentage-metric: "vllm:kv_cache_usage_perc"
  image:
    name: llm-d-inference-scheduler
    hub: ghcr.io/llm-d
    tag: v0.5.0
    pullPolicy: Always
  extProcPort: 9002
  pluginsConfigFile: "default-plugins.yaml"
  monitoring:
    interval: "10s"
    secret:
      name: <deployment-name>-gateway-sa-metrics-reader-secret  # Replace with actual deployment name
    prometheus:
      enabled: true  # Replace <monitoring-enabled> with true or false
      auth:
        enabled: true
inferencePool:
  targetPorts:
    - number: 8000
  modelServerType: vllm
  modelServers:
    matchLabels:
      llm-d.ai/inference-serving: "true"
      llm-d.ai/deployment: "<deployment-name>"  # MUST match ms-values.yaml labels
```

#### Generate ms-values.yaml:

**CRITICAL**: The `labels` in this file MUST exactly match the `matchLabels` in gaie-values.yaml.

```yaml
multinode: false  # Set to true if tensor parallelism > 1

modelArtifacts:
  uri: "hf://Qwen/Qwen3-32B"  # Replace with actual model URI
  name: "Qwen/Qwen3-32B"  # Replace with actual model name
  size: 80Gi  # Replace with actual model size
  authSecretName: "llm-d-hf-token"
  labels:
    llm-d.ai/inference-serving: "true"
    llm-d.ai/deployment: "custom-deployment"  # MUST match gaie-values.yaml matchLabels
    llm-d.ai/accelerator-variant: "gpu"  # "gpu" for nvidia/amd, "xpu" for intel, "hpu" for habana, "tpu" for tpu, "cpu" for cpu
    llm-d.ai/accelerator-vendor: "nvidia"  # "nvidia", "amd", "intel", "habana", "google", "cpu"
    llm-d.ai/model: "Qwen3-32B"  # Replace with model name (without org prefix)

routing:
  proxy:
    enabled: false  # Set to true only if prefill/decode disaggregation is enabled
    targetPort: 8000

accelerator:
  type: nvidia  # Replace with: nvidia | amd | intel-i915 | intel-xe | habana | tpu | cpu

decode:
  create: true
  parallelism:
    tensor: 2  # Replace with actual tensor parallelism value
    data: 1
  replicas: 8  # Replace with actual number of replicas
  monitoring:
    podmonitor:
      enabled: true  # Replace with true or false
      portName: "vllm"
      path: "/metrics"
      interval: "30s"
  containers:
    - name: "vllm"
      image: ghcr.io/llm-d/llm-d-cuda:v0.5.0  # Auto-select based on accelerator: cuda|rocm|xpu|hpu|tpu|cpu
      modelCommand: vllmServe
      args:
        - "--disable-uvicorn-access-log"
        - "--gpu-memory-utilization=0.95"  # Replace with actual value (0.0-1.0)
        # Add additional args as needed, e.g.:
        # - "--max-model-len=4096"
      ports:
        - containerPort: 8000
          name: vllm
          protocol: TCP
      resources:
        limits:
          cpu: '32'  # Replace with actual CPU limit
          memory: 100Gi  # Replace with actual memory limit
        requests:
          cpu: '32'  # Replace with actual CPU request
          memory: 100Gi  # Replace with actual memory request
      mountModelVolume: true
      volumeMounts:
        - name: metrics-volume
          mountPath: /.config
        - name: shm
          mountPath: /dev/shm
        - name: torch-compile-cache
          mountPath: /.cache
      startupProbe:
        httpGet:
          path: /v1/models
          port: vllm
        initialDelaySeconds: 15
        periodSeconds: 30
        timeoutSeconds: 5
        failureThreshold: 60
      livenessProbe:
        httpGet:
          path: /health
          port: vllm
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /v1/models
          port: vllm
        periodSeconds: 5
        timeoutSeconds: 2
        failureThreshold: 3
  volumes:
    - name: metrics-volume
      emptyDir: {}
    - name: torch-compile-cache
      emptyDir: {}
    - name: shm
      emptyDir:
        medium: Memory
        sizeLimit: 20Gi  # Replace with actual shm size (typically 20-40Gi)

prefill:
  create: false  # Set to true only if prefill/decode disaggregation is enabled
  # If enabled, copy decode configuration and adjust replicas/resources as needed
```

**Image Selection by Accelerator Type**:
- `nvidia` → `ghcr.io/llm-d/llm-d-cuda:v0.5.0`
- `amd` → `ghcr.io/llm-d/llm-d-rocm:v0.5.0`
- `intel-i915` or `intel-xe` → `ghcr.io/llm-d/llm-d-xpu:v0.5.0`
- `habana` → `ghcr.io/llm-d/llm-d-hpu:v0.5.0`
- `tpu` → `ghcr.io/llm-d/llm-d-tpu:v0.5.0`
- `cpu` → `ghcr.io/llm-d/llm-d-cpu:v0.5.0`

### Step 4: Generate httproute.yaml

Create an HTTPRoute configuration:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <deployment-name>
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: infra-<deployment-name>-inference-gateway
  rules:
    - backendRefs:
        - group: inference.networking.k8s.io
          kind: InferencePool
          name: gaie-<deployment-name>
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


### Step 5: Create Deployment Directory and Generate Files

**TOOL USAGE INSTRUCTIONS**: Follow these exact steps using the specified tools.

#### 5.1 Create Deployment Directory Structure

Use `execute_command` to create the directory structure:

```bash
# Create main deployment directory and subdirectories
mkdir -p <deployment-directory>/<deployment-name>
```

**Example command**:
```bash
mkdir -p ./custom-llama-deploy/custom-llama
```

#### 5.2 Write Generated Files

Use `write_to_file` tool to create each NEW configuration file. These are NEW files, not modifications to existing files.

**CRITICAL - Label Validation**: Before writing files, verify that:
- Labels in gaie-values.yaml `matchLabels` section
- EXACTLY MATCH labels in ms-values.yaml `modelArtifacts.labels` section
- Specifically: `llm-d.ai/deployment` value must be identical in both files

**Write files in this order**:

1. **Write helmfile.yaml**:
   - Tool: `write_to_file`
   - Path: `<deployment-directory>/helmfile.yaml`
   - Content: The generated helmfile configuration from Step 2 with ALL placeholders replaced

2. **Write gaie-values.yaml**:
   - Tool: `write_to_file`
   - Path: `<deployment-directory>/<deployment-name>/gaie-values.yaml`
   - Content: The generated GAIE configuration from Step 3 with ALL placeholders replaced

3. **Write ms-values.yaml**:
   - Tool: `write_to_file`
   - Path: `<deployment-directory>/<deployment-name>/ms-values.yaml`
   - Content: The generated model server configuration from Step 3 with ALL placeholders replaced

4. **Write httproute.yaml**:
   - Tool: `write_to_file`
   - Path: `<deployment-directory>/httproute.yaml`
   - Content: The generated HTTPRoute configuration from Step 4 with ALL placeholders replaced

5. **Write README.md**:
   - Tool: `write_to_file`
   - Path: `<deployment-directory>/README.md`
   - Content: Concise deployment documentation including:
     - Configuration summary (model, hardware, namespace)
     - Quick deployment commands with actual namespace and deployment name
     - Verification steps
     - Gateway address retrieval command
     - Basic inference test example with actual model name

**After writing all files, inform the user**:
```
Created deployment files in <deployment-directory>:
├── helmfile.yaml
├── httproute.yaml
├── README.md
└── <deployment-name>/
    ├── gaie-values.yaml
    └── ms-values.yaml

All placeholders have been replaced with actual values.
Ready for deployment.
```

### Step 6: Execute Deployment

After generating all configuration files, execute the deployment:

#### 6.1 Verify Prerequisites

Before deploying, verify all prerequisites are met:

```bash
# Check if namespace exists, create if not
kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}

# Verify HuggingFace token secret exists
kubectl get secret llm-d-hf-token -n ${NAMESPACE} || echo "WARNING: HuggingFace token secret not found"

# Check for Gateway API CRDs
kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null || echo "WARNING: Gateway API CRDs not installed"

# Check for Inference Extension CRDs
kubectl get crd inferencepools.inference.networking.k8s.io 2>/dev/null || echo "WARNING: Inference Extension CRDs not installed"
```

#### 6.2 Navigate to Deployment Directory

```bash
cd <deployment-directory>
```

#### 6.3 Deploy Using Helmfile

Execute the helmfile deployment:

```bash
export NAMESPACE=<namespace>
helmfile apply -n ${NAMESPACE}
```

**Monitor the deployment progress:**

```bash
# Watch pods being created
kubectl get pods -n ${NAMESPACE} -w
```

**Wait for all pods to be running:**

```bash
# Check pod status
kubectl get pods -n ${NAMESPACE}

# If pods are not running, check events
kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp'
```

#### 6.4 Apply HTTPRoute

Once the helmfile deployment is complete and pods are running, apply the HTTPRoute:

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

**Verify HTTPRoute is accepted:**

```bash
kubectl get httproute -n ${NAMESPACE}
kubectl describe httproute <deployment-name> -n ${NAMESPACE}
```

#### 6.5 Validate Deployment

Perform comprehensive validation:

**Check all resources:**

```bash
# Check InferencePool status
kubectl get inferencepool -n ${NAMESPACE}
kubectl describe inferencepool -n ${NAMESPACE}

# Check Gateway status
kubectl get gateway -n ${NAMESPACE}
kubectl describe gateway -n ${NAMESPACE}

# Check Services
kubectl get svc -n ${NAMESPACE}

# Check PVCs (if applicable)
kubectl get pvc -n ${NAMESPACE}
```

**Verify pod health:**

```bash
# Check all pods are running
kubectl get pods -n ${NAMESPACE}

# Check pod logs for errors
kubectl logs -l llm-d.ai/deployment=<deployment-name> -n ${NAMESPACE} --tail=50

# Check for any CrashLoopBackOff or ImagePullBackOff
kubectl get pods -n ${NAMESPACE} | grep -E "CrashLoop|ImagePull|Error"
```

**Test connectivity:**

```bash
# Get Gateway address
GATEWAY_ADDRESS=$(kubectl get gateway -n ${NAMESPACE} -o jsonpath='{.items[0].status.addresses[0].value}')
echo "Gateway Address: ${GATEWAY_ADDRESS}"

# Test health endpoint
curl -v http://${GATEWAY_ADDRESS}/health

# Test models endpoint
curl http://${GATEWAY_ADDRESS}/v1/models
```

#### 6.6 Wait for Model Loading

Model loading can take several minutes depending on model size:

```bash
# Monitor vLLM logs for model loading progress
kubectl logs -l llm-d.ai/deployment=<deployment-name> -n ${NAMESPACE} -f | grep -E "Loading|model|ready"
```

**Expected log messages:**
- "Loading model weights..."
- "Model loaded successfully"
- "vLLM API server started"

#### 6.7 Perform Inference Test

Once the model is loaded, test inference:

```bash
# Simple completion test
curl -X POST http://${GATEWAY_ADDRESS}/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<model-name>",
    "prompt": "Hello, how are you?",
    "max_tokens": 50,
    "temperature": 0.7
  }'
```

**Expected response:**
```json
{
  "id": "cmpl-...",
  "object": "text_completion",
  "created": 1234567890,
  "model": "<model-name>",
  "choices": [
    {
      "text": "...",
      "index": 0,
      "finish_reason": "length"
    }
  ]
}
```

#### 6.9 Common Deployment Issues and Fixes

**ERROR HANDLING INSTRUCTIONS**: When encountering these issues, take the specified action.

| Issue | Diagnosis Command | Agent Action |
|-------|------------------|--------------|
| Pods stuck in Pending | `kubectl describe pod <pod-name> -n ${NAMESPACE}` | Check events for "Insufficient" messages. If insufficient GPU/CPU/memory, use `ask_followup_question` to inform user about resource constraints and ask if they want to adjust requirements or add nodes. |
| ImagePullBackOff | `kubectl describe pod <pod-name> -n ${NAMESPACE}` | Check image name and tag. If image doesn't exist, verify accelerator type selection. Use `ask_followup_question` to confirm correct accelerator type. |
| CrashLoopBackOff | `kubectl logs <pod-name> -n ${NAMESPACE}` | Check logs for error message. If "Out of memory", suggest reducing gpu-memory-utilization. If "model not found", verify model URI. Use `ask_followup_question` with specific error and suggested fix. |
| Model download fails | `kubectl logs <pod-name> -n ${NAMESPACE}` | Check for "401 Unauthorized" or "403 Forbidden". If found, HuggingFace token is invalid/missing. STOP and use `ask_followup_question` to request token verification. |
| Gateway not routing | `kubectl describe httproute -n ${NAMESPACE}` | Check HTTPRoute conditions. If "Backend not found", InferencePool may not be ready. Wait 30s and recheck. If still failing, verify label matching between gaie-values.yaml and ms-values.yaml. |
| Out of memory (OOM) | `kubectl logs <pod-name> -n ${NAMESPACE}` | If OOM error found, calculate: current_memory * 1.5 = new_memory. Use `ask_followup_question` to suggest increasing memory limits or reducing gpu-memory-utilization to 0.85. |
| InferencePool not Ready | `kubectl describe inferencepool -n ${NAMESPACE}` | Check conditions. If "No backends available", model servers may not have matching labels. Verify labels match between gaie-values.yaml and ms-values.yaml. If mismatch found, STOP and inform user of label mismatch issue. |

#### 6.10 Deployment Success Criteria

**MANDATORY**: Before using `attempt_completion`, verify ALL criteria are met.

Use `execute_command` to run final validation:

```bash
echo "=== Deployment Validation ==="
echo "Pods:"
kubectl get pods -n ${NAMESPACE}
echo ""
echo "InferencePool:"
kubectl get inferencepool -n ${NAMESPACE}
echo ""
echo "Gateway:"
kubectl get gateway -n ${NAMESPACE}
echo ""
echo "HTTPRoute:"
kubectl get httproute -n ${NAMESPACE}
echo ""
echo "Gateway Address: $(kubectl get gateway -n ${NAMESPACE} -o jsonpath='{.items[0].status.addresses[0].value}')"
```

**Check each criterion**:

1. ✅ All pods show `Running` status with `Ready` column showing `1/1` or `2/2`
2. ✅ InferencePool shows `Ready: True` condition
3. ✅ Gateway shows `Programmed: True` and `Accepted: True` conditions
4. ✅ HTTPRoute shows `Accepted: True` condition
5. ✅ Model loading logs show "vLLM API server started"
6. ✅ Health endpoint (`curl http://${GATEWAY_ADDRESS}/health`) returns 200 OK
7. ✅ Models endpoint (`curl http://${GATEWAY_ADDRESS}/v1/models`) lists the deployed model
8. ✅ Inference test returns valid JSON completion response

**IF ANY CRITERION FAILS**:
- DO NOT use `attempt_completion`
- Diagnose the failure using section 6.9
- Fix the issue
- Re-validate all criteria

**ONLY use `attempt_completion` when ALL 8 criteria pass.**

### Step 7: Provide Deployment Summary

**After ALL 8 success criteria pass**, use `attempt_completion` with this format:

```
Deployment complete.

Configuration:
- Name: <deployment-name>
- Namespace: <namespace>
- Model: <model-name>
- Accelerator: <accelerator-type>
- Gateway: http://<gateway-address>

Files: <deployment-directory>/
- helmfile.yaml
- httproute.yaml
- README.md
- <deployment-name>/gaie-values.yaml
- <deployment-name>/ms-values.yaml

Monitoring:
kubectl logs -l llm-d.ai/deployment=<deployment-name> -n <namespace>
kubectl get pods,inferencepool,gateway,httproute -n <namespace>
```

**Do NOT include**:
- Questions or offers for assistance
- Phrases like "Great!" or "Certainly"
- Requests for feedback

---

## Quick Reference

### Image Selection by Accelerator
| Accelerator | Image |
|-------------|-------|
| `nvidia` | `ghcr.io/llm-d/llm-d-cuda:v0.5.0` |
| `amd` | `ghcr.io/llm-d/llm-d-rocm:v0.5.0` |
| `intel-i915` / `intel-xe` | `ghcr.io/llm-d/llm-d-xpu:v0.5.0` |
| `habana` | `ghcr.io/llm-d/llm-d-hpu:v0.5.0` |
| `tpu` | `ghcr.io/llm-d/llm-d-tpu:v0.5.0` |
| `cpu` | `ghcr.io/llm-d/llm-d-cpu:v0.5.0` |

**Reference Examples**:
- [NVIDIA values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values.yaml)
- [AMD values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_amd.yaml)
- [Intel XPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_xpu.yaml)
- [Intel Gaudi values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values-hpu.yaml)
- [TPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_tpu.yaml)
- [CPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_cpu.yaml)

### Gateway Provider Options
| Provider | Use Case | Configuration |
|----------|----------|---------------|
| `istio` | Default, full-featured | [Istio config](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/istio.yaml) |
| `kgateway` | Lightweight, simple | [K-Gateway config](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/kgateway.yaml) |
| `agentgateway` | Inference-optimized | [Agent Gateway config](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/agentgateway.yaml) |
| `gke` | Google Cloud managed | [GKE config](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/gke.yaml) |

---

## Additional Resources

### Documentation
- [llm-d Project](https://github.com/llm-d/llm-d)
- [Project Overview](https://github.com/llm-d/llm-d/blob/main/PROJECT.md)
- [Well-lit Paths](https://github.com/llm-d/llm-d/blob/main/guides/README.md)
- [Quickstart Guide](https://github.com/llm-d/llm-d/blob/main/guides/QUICKSTART.md)
- [Gateway Customization](https://github.com/llm-d/llm-d/blob/main/docs/customizing-your-gateway.md)

### External Resources
- [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)
- [vLLM Documentation](https://docs.vllm.ai)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io)

### Helm Chart Repositories
- **llm-d-infra**: `https://llm-d-incubation.github.io/llm-d-infra/` - Infrastructure components
- **llm-d-modelservice**: `https://llm-d-incubation.github.io/llm-d-modelservice/` - Model server
- **inferencepool**: `oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool` - Inference scheduler

### CRD Installation Commands
```bash
# Gateway API CRDs (v1.4.0)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Inference Extension CRDs (v1.3.0)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.3.0/install.yaml
```