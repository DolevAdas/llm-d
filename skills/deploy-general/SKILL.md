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

Ask the user for the following information (provide intelligent defaults based on common configurations):

#### Required Information:
1. **Deployment Name** - A unique name for this deployment (e.g., "custom-llama-deployment")
2. **Target Namespace** - Kubernetes namespace for deployment (default: "llmd-custom")
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

Based on the gathered requirements, generate a `helmfile.yaml` file with the following structure.

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

5. **Write README.md**:
   - Location: `<deployment-directory>/README.md`
   - Content: Concise deployment documentation including:
     - Configuration summary (model, hardware, namespace)
     - Quick deployment commands
     - Verification steps
     - Gateway address retrieval
     - Basic inference test example

**Inform the user:**
```
Created deployment files in <deployment-directory>:
├── helmfile.yaml
├── httproute.yaml
├── README.md
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
3. **Next steps**: Link to README.md for detailed usage
4. **Monitoring**: How to access metrics and logs
5. **Troubleshooting**: Common issues and solutions


## Configuration Guidelines

### Hardware-Specific Configurations

**Reference**: [llm-d Accelerator Documentation](https://github.com/llm-d/llm-d/blob/main/docs/accelerators/README.md)

#### NVIDIA GPUs
```yaml
accelerator:
  type: nvidia
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-cuda:v0.5.0
```
**Example**: [NVIDIA GPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values.yaml)

#### AMD GPUs
```yaml
accelerator:
  type: amd
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-rocm:v0.5.0
```
**Example**: [AMD GPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_amd.yaml)

#### Intel XPU
```yaml
accelerator:
  type: intel-i915  # or intel-xe for BMG
  dra: true
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-xpu:v0.5.0
```
**Example**: [Intel XPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_xpu.yaml)

#### Intel Gaudi (HPU)
```yaml
accelerator:
  type: habana
  dra: true
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-hpu:v0.5.0
```
**Example**: [Intel Gaudi values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values-hpu.yaml)

#### Google TPU
```yaml
accelerator:
  type: tpu
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-tpu:v0.5.0
```
**Example**: [TPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_tpu.yaml)

#### CPU
```yaml
accelerator:
  type: cpu
decode:
  containers:
    - image: ghcr.io/llm-d/llm-d-cpu:v0.5.0
```
**Example**: [CPU values](https://github.com/llm-d/llm-d/blob/main/guides/inference-scheduling/ms-inference-scheduling/values_cpu.yaml)

### Gateway-Specific Configurations

**Reference**: [Gateway Provider Setup](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/README.md)

#### Istio
- Uses DestinationRule for connection pooling
- Supports TLS configuration
- Default choice for most deployments
- **Configuration**: [Istio values](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/istio.yaml)

#### K-Gateway
- Lightweight alternative to Istio
- Good for simpler deployments
- Less operational overhead
- **Configuration**: [K-Gateway values](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/kgateway.yaml)

#### Agent Gateway
- Specialized for inference workloads
- Advanced routing capabilities
- **Configuration**: [Agent Gateway values](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/agentgateway.yaml)

#### GKE
- Managed by Google Cloud
- Automatic load balancer provisioning
- Regional internal or external options
- **Configuration**: [GKE values](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/common-configurations/gke.yaml)

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



## Additional Resources

- **llm-d Project**: [https://github.com/llm-d/llm-d](https://github.com/llm-d/llm-d)
- **llm-d Documentation**: [https://llm-d.ai](https://llm-d.ai)
- **Project Overview**: [PROJECT.md](https://github.com/llm-d/llm-d/blob/main/PROJECT.md)
- **Well-lit Paths**: [guides/README.md](https://github.com/llm-d/llm-d/blob/main/guides/README.md)
- **Quickstart Guide**: [guides/QUICKSTART.md](https://github.com/llm-d/llm-d/blob/main/guides/QUICKSTART.md)
- **Gateway Customization**: [docs/customizing-your-gateway.md](https://github.com/llm-d/llm-d/blob/main/docs/customizing-your-gateway.md)
- **Inference Gateway**: [https://github.com/kubernetes-sigs/gateway-api-inference-extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)
- **vLLM Documentation**: [https://docs.vllm.ai](https://docs.vllm.ai)
- **Gateway API Documentation**: [https://gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io)

## Helm Chart Repositories

The skill uses the following Helm chart repositories (no local clone required):

- **llm-d-infra**: `https://llm-d-incubation.github.io/llm-d-infra/`
  - Infrastructure components (Gateway, monitoring)
  - [Chart Documentation](https://github.com/llm-d-incubation/llm-d-infra)

- **llm-d-modelservice**: `https://llm-d-incubation.github.io/llm-d-modelservice/`
  - Model server deployment
  - [Chart Documentation](https://github.com/llm-d-incubation/llm-d-modelservice)

- **inferencepool**: `oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool`
  - Inference scheduler (EPP)
  - [Chart Documentation](https://github.com/kubernetes-sigs/gateway-api-inference-extension)

## CRD Installation

Before using this skill, ensure Gateway API and Inference Extension CRDs are installed:

```bash
# Install Gateway API CRDs (v1.4.0)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Install Inference Extension CRDs (v1.3.0)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/v1.3.0/install.yaml
```

**Reference**: [Gateway Provider Prerequisites](https://github.com/llm-d/llm-d/blob/main/guides/prereq/gateway-provider/README.md)