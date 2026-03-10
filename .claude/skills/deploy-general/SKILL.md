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

1. **LLMD_PATH Environment Variable** - Point to your llm-d repository clone
2. **Client Tools** - kubectl, helm, helmfile, git (see `guides/prereq/client-setup/README.md`)
3. **HuggingFace Token** - Secret `llm-d-hf-token` with key `HF_TOKEN` in target namespace
4. **Gateway Provider** - Istio, K-Gateway, or Agent Gateway (see `guides/prereq/gateway-provider/README.md`)
5. **Infrastructure** - Sufficient cluster resources (see `guides/prereq/infrastructure/README.md`)
6. **Target Directory** - A directory where deployment files will be generated

## Deployment Workflow

When a user requests a custom llm-d deployment, follow this workflow:

### Step 1: Gather Deployment Requirements

Ask the user for the following information (provide intelligent defaults based on common configurations):

#### Required Information:
1. **Deployment Name** - A unique name for this deployment (e.g., "custom-llama-deployment")
2. **Target Namespace** - Kubernetes namespace for deployment (default: "llm-d-custom")
3. **Model Configuration**:
   - Model URI (HuggingFace model ID or path)
   - Model name
   - Model size (storage requirement)
4. **Hardware Configuration**:
   - Accelerator type (nvidia, amd, xpu, hpu, tpu, cpu)
   - Number of GPUs per replica (tensor parallelism)
   - Number of replicas (data parallelism)
5. **Gateway Provider**:
   - Gateway type (istio, kgateway, agentgateway, gke)

#### Optional Information:
1. **Advanced vLLM Settings**:
   - GPU memory utilization (default: 0.95)
   - Additional vLLM arguments
   - Custom container image
2. **Resource Limits**:
   - CPU limits/requests
   - Memory limits/requests
3. **Prefill/Decode Configuration**:
   - Enable disaggregation (yes/no)
   - Prefill replicas
   - Decode replicas
4. **Storage Configuration**:
   - Storage class
   - PVC size
5. **Monitoring**:
   - Enable Prometheus monitoring (default: true)

### Step 2: Generate helmfile.yaml

Based on the gathered requirements, generate a `helmfile.yaml` file with the following structure:

```yaml
environments:
  <gateway-type>:
    values:
      - ../prereq/gateway-provider/common-configurations/<gateway-type>.yaml
  default:
    values:
      - ../prereq/gateway-provider/common-configurations/<gateway-type>.yaml

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
- Replace `<gateway-type>` with the selected gateway provider
- Replace `<namespace>` with the target namespace
- Replace `<deployment-name>` with the deployment name
- Adjust chart versions to match current llm-d release

### Step 3: Generate Supporting Values Files

Create a directory structure for the deployment:
```
<deployment-name>/
├── helmfile.yaml
├── httproute.yaml
├── README.md
├── gaie-values.yaml
└── ms-values.yaml
```

#### Generate gaie-values.yaml:

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
      name: <deployment-name>-gateway-sa-metrics-reader-secret
    prometheus:
      enabled: <monitoring-enabled>
      auth:
        enabled: true
inferencePool:
  targetPorts:
    - number: 8000
  modelServerType: vllm
  modelServers:
    matchLabels:
      llm-d.ai/inference-serving: "true"
      llm-d.ai/deployment: "<deployment-name>"
```

#### Generate ms-values.yaml:

```yaml
multinode: <true if tensor parallelism > 1>

modelArtifacts:
  uri: "<model-uri>"
  name: "<model-name>"
  size: <model-size>
  authSecretName: "llm-d-hf-token"
  labels:
    llm-d.ai/inference-serving: "true"
    llm-d.ai/deployment: "<deployment-name>"
    llm-d.ai/accelerator-variant: "<accelerator-variant>"
    llm-d.ai/accelerator-vendor: "<accelerator-vendor>"
    llm-d.ai/model: "<model-name>"

routing:
  proxy:
    enabled: <true if prefill/decode disaggregation>
    targetPort: 8000

accelerator:
  type: <accelerator-type>

decode:
  create: true
  parallelism:
    tensor: <tensor-parallelism>
    data: 1
  replicas: <replicas>
  monitoring:
    podmonitor:
      enabled: <monitoring-enabled>
      portName: "vllm"
      path: "/metrics"
      interval: "30s"
  containers:
    - name: "vllm"
      image: <vllm-image>
      modelCommand: vllmServe
      args:
        - "--disable-uvicorn-access-log"
        - "--gpu-memory-utilization=<gpu-memory-utilization>"
        <additional-args>
      ports:
        - containerPort: 8000
          name: vllm
          protocol: TCP
      resources:
        limits:
          cpu: '<cpu-limit>'
          memory: <memory-limit>
        requests:
          cpu: '<cpu-request>'
          memory: <memory-request>
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
        sizeLimit: <shm-size>

prefill:
  create: <true if prefill/decode disaggregation>
  <prefill-configuration if enabled>
```

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

### Step 5: Generate README.md

Create a comprehensive README documenting the deployment:

```markdown
# <Deployment Name> - llm-d Deployment

## Overview

This deployment was generated using the llm-d general deployment skill. It configures llm-d to serve the <model-name> model on <accelerator-type> hardware.

## Configuration Summary

- **Deployment Name**: <deployment-name>
- **Namespace**: <namespace>
- **Model**: <model-name> (<model-uri>)
- **Hardware**: <accelerator-type>
- **Tensor Parallelism**: <tensor-parallelism>
- **Replicas**: <replicas>
- **Gateway Provider**: <gateway-type>
- **Prefill/Decode Disaggregation**: <enabled/disabled>

## Prerequisites

Before deploying, ensure:

1. Kubernetes cluster with sufficient resources
2. Gateway provider (<gateway-type>) installed and configured
3. HuggingFace token secret created:
   ```bash
   kubectl create secret generic llm-d-hf-token \
     --from-literal=HF_TOKEN=<your-token> \
     -n <namespace>
   ```
4. Namespace created:
   ```bash
   kubectl create namespace <namespace>
   ```

## Deployment Instructions

### 1. Navigate to deployment directory

```bash
cd <deployment-directory>
```

### 2. Deploy using helmfile

```bash
export NAMESPACE=<namespace>
helmfile apply -n ${NAMESPACE}
```

### 3. Apply HTTPRoute

```bash
kubectl apply -f httproute.yaml -n ${NAMESPACE}
```

### 4. Verify deployment

```bash
# Check pods
kubectl get pods -n ${NAMESPACE}

# Check InferencePool
kubectl get inferencepool -n ${NAMESPACE}

# Check Gateway
kubectl get gateway -n ${NAMESPACE}

# Check HTTPRoute
kubectl get httproute -n ${NAMESPACE}
```

## Testing the Deployment

### Get Gateway Address

```bash
kubectl get gateway -n ${NAMESPACE} -o jsonpath='{.items[0].status.addresses[0].value}'
```

### Test Inference

```bash
curl -X POST http://<gateway-address>/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "<model-name>",
    "prompt": "Hello, how are you?",
    "max_tokens": 50
  }'
```

## Resource Requirements

- **GPUs**: <total-gpus> x <accelerator-type>
- **CPU**: <total-cpu>
- **Memory**: <total-memory>
- **Storage**: <model-size>

## Customization

To modify this deployment:

1. Edit `ms-values.yaml` for model server configuration
2. Edit `gaie-values.yaml` for inference scheduler configuration
3. Edit `helmfile.yaml` for overall deployment structure
4. Reapply: `helmfile apply -n ${NAMESPACE}`

## Monitoring

<If monitoring enabled>
Prometheus metrics are exposed at:
- vLLM metrics: `http://<pod-ip>:8000/metrics`
- Inference scheduler metrics: `http://<epp-pod-ip>:9002/metrics`

## Troubleshooting

### Pods not starting

```bash
kubectl describe pod <pod-name> -n ${NAMESPACE}
kubectl logs <pod-name> -n ${NAMESPACE}
```

### Model not loading

Check vLLM logs:
```bash
kubectl logs -l llm-d.ai/deployment=<deployment-name> -n ${NAMESPACE}
```

### Gateway not routing

```bash
kubectl describe gateway -n ${NAMESPACE}
kubectl describe httproute -n ${NAMESPACE}
```

## Cleanup

To remove this deployment:

```bash
helmfile destroy -n ${NAMESPACE}
kubectl delete httproute <deployment-name> -n ${NAMESPACE}
```

## Additional Resources

- [llm-d Documentation](https://llm-d.ai)
- [vLLM Documentation](https://docs.vllm.ai)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io)
```

### Step 5: Create Deployment Directory and Generate Files

Before deployment, create the directory structure and write all configuration files:

#### 5.1 Create Deployment Directory Structure

```bash
# Create main deployment directory
mkdir -p <deployment-directory>

# Create subdirectories for values files
mkdir -p <deployment-directory>/<deployment-name>
```

#### 5.2 Write Generated Files

Write all the generated configuration files to the deployment directory:

1. **Write helmfile.yaml**:
   - Location: `<deployment-directory>/helmfile.yaml`
   - Content: The generated helmfile configuration from Step 2

2. **Write gaie-values.yaml**:
   - Location: `<deployment-directory>/<deployment-name>/gaie-values.yaml`
   - Content: The generated GAIE configuration from Step 3

3. **Write ms-values.yaml**:
   - Location: `<deployment-directory>/<deployment-name>/ms-values.yaml`
   - Content: The generated model server configuration from Step 3

4. **Write httproute.yaml**:
   - Location: `<deployment-directory>/httproute.yaml`
   - Content: The generated HTTPRoute configuration from Step 4

5. **Write HELPFILE.md**:
   - Location: `<deployment-directory>/HELPFILE.md`
   - Content: The generated helpfile documentation from Step 5

**Inform the user:**
```
Created deployment files in <deployment-directory>:
├── helmfile.yaml
├── httproute.yaml
├── HELPFILE.md
└── <deployment-name>/
    ├── gaie-values.yaml
    └── ms-values.yaml
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

#### 6.8 Common Deployment Issues and Fixes

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| Pods stuck in Pending | `kubectl describe pod <pod-name> -n ${NAMESPACE}` | Check node resources, GPU availability, and node selectors |
| ImagePullBackOff | `kubectl describe pod <pod-name> -n ${NAMESPACE}` | Verify image name, tag, and registry credentials |
| CrashLoopBackOff | `kubectl logs <pod-name> -n ${NAMESPACE}` | Check vLLM logs for errors, verify model URI and token |
| Model download fails | `kubectl logs <pod-name> -n ${NAMESPACE} \| grep download` | Verify HuggingFace token and network connectivity |
| Gateway not routing | `kubectl describe httproute -n ${NAMESPACE}` | Verify Gateway and InferencePool are ready |
| Out of memory | `kubectl logs <pod-name> -n ${NAMESPACE} \| grep -i memory` | Reduce GPU memory utilization or increase memory limits |

#### 6.9 Deployment Success Criteria

The deployment is successful when:

1. ✅ All pods are in `Running` state with `Ready` status
2. ✅ InferencePool shows `Ready` condition
3. ✅ Gateway shows `Programmed` and `Accepted` conditions
4. ✅ HTTPRoute shows `Accepted` condition
5. ✅ Model loading logs show successful completion
6. ✅ Health endpoint returns 200 OK
7. ✅ Models endpoint lists the deployed model
8. ✅ Inference test returns valid completion

**Final validation command:**

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

### Step 7: Provide Deployment Summary

After successful deployment, provide the user with:

1. **Deployment location**: Path to the deployment directory
2. **Access information**: Gateway address and endpoints
3. **Next steps**: Link to HELPFILE.md for detailed usage
4. **Monitoring**: How to access metrics and logs
5. **Troubleshooting**: Common issues and solutions

### Step 6: Create Deployment Directory and Files

1. Create the deployment directory structure
2. Write all generated files to the directory
3. Inform the user of the created files and their locations
4. Provide next steps for deployment

## Configuration Guidelines

### Hardware-Specific Configurations

#### NVIDIA GPUs
```yaml
accelerator:
  type: nvidia
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-cuda:v0.5.0
```

#### AMD GPUs
```yaml
accelerator:
  type: amd
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-rocm:v0.5.0
```

#### Intel XPU
```yaml
accelerator:
  type: intel-i915  # or intel-xe for BMG
  dra: true
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-xpu:v0.5.0
```

#### Intel Gaudi (HPU)
```yaml
accelerator:
  type: habana
  dra: true
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-hpu:v0.5.0
```

#### Google TPU
```yaml
accelerator:
  type: tpu
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-tpu:v0.5.0
```

#### CPU
```yaml
accelerator:
  type: cpu
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-cpu:v0.5.0
```

### Gateway-Specific Configurations

#### Istio
- Uses DestinationRule for connection pooling
- Supports TLS configuration
- Default choice for most deployments

#### K-Gateway
- Lightweight alternative to Istio
- Good for simpler deployments
- Less operational overhead

#### Agent Gateway
- Specialized for inference workloads
- Advanced routing capabilities

#### GKE
- Managed by Google Cloud
- Automatic load balancer provisioning
- Regional internal or external options

## Best Practices

1. **Naming Conventions**:
   - Use short, descriptive deployment names
   - Avoid special characters except hyphens
   - Keep namespace names under 20 characters

2. **Resource Sizing**:
   - Start with conservative resource limits
   - Monitor actual usage and adjust
   - Leave headroom for spikes

3. **Model Storage**:
   - Use appropriate storage class for your cloud provider
   - Consider model size + 20% for overhead
   - Enable caching for faster restarts

4. **Monitoring**:
   - Always enable Prometheus monitoring
   - Set up alerts for key metrics
   - Monitor GPU utilization

5. **Security**:
   - Use secrets for sensitive data
   - Enable RBAC
   - Consider network policies

## Common Issues and Solutions

### Issue: Pods stuck in Pending
**Solution**: Check node resources and GPU availability
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Issue: Model download fails
**Solution**: Verify HuggingFace token and network connectivity
```bash
kubectl logs <pod-name> -n ${NAMESPACE} | grep -i "download\|token"
```

### Issue: Gateway not routing traffic
**Solution**: Verify HTTPRoute and Gateway configuration
```bash
kubectl describe httproute -n ${NAMESPACE}
kubectl describe gateway -n ${NAMESPACE}
```

### Issue: Out of memory errors
**Solution**: Increase memory limits or reduce GPU memory utilization
```yaml
args:
  - "--gpu-memory-utilization=0.85"  # Reduce from 0.95
```

# Additional Resources

- **llm-d Project**: [PROJECT.md](../../PROJECT.md)
- **Well-lit Paths**: [guides/README.md](../../guides/README.md)
- **Gateway Customization**: [docs/customizing-your-gateway.md](../../docs/customizing-your-gateway.md)
- **Monitoring Setup**: [docs/monitoring/README.md](../../docs/monitoring/README.md)