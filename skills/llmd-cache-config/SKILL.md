---
name: llmd-cache-config
description: Modify cache memory settings in existing llm-d deployments without full redeployment. Adjust GPU memory utilization, KV cache capacity, shared memory, block size, and context length to optimize performance for different workload patterns. Use when you need to tune cache settings, increase throughput, reduce latency, or support longer contexts.
---

# llm-d Cache Configuration Skill

## 📋 Command Execution Notice

**Before executing any command, I will:**
1. **Explain what the command does** - Clear description of purpose and expected outcome
2. **Show the actual command** - The exact command to be executed
3. **Explain why it's needed** - How it fits into the workflow

> ## 🔔 ALWAYS NOTIFY BEFORE CREATING RESOURCES
>
> **RULE**: Before creating ANY resource (namespaces, files, Kubernetes objects), notify the user first.
>
> **Format**: "I am about to create `<resource-type>` named `<name>` because `<reason>`. Proceeding now."
>
> **Never silently create resources.** Check existence first, then notify before acting.

## Critical Rules

1. **Do NOT change cluster-level definitions** - All changes must be within the designated namespace. Never modify cluster-wide resources. Always scope commands with `-n ${NAMESPACE}`.

2. **Do NOT modify existing repository code** - Only create new files. Never edit pre-existing repository files.

3. **Script modifications** - If existing scripts need updates, copy them to your deployment directory and modify the copy. Never edit scripts in `skills/llmd-cache-config/scripts/` directly.

## Overview

Modify cache settings in existing llm-d deployments: GPU memory utilization, block size, max context length, and shared memory (SHM). Changes apply via rolling updates with automatic backups.

**For deployments with CPU offloading already enabled**: You can also tune CPU cache size and InferencePool prefix cache scorer configurations.

**Note**: Initial setup of tiered prefix cache offloading (CPU RAM, local disk, or shared storage) requires redeployment. See [`guides/tiered-prefix-cache/README.md`](../../guides/tiered-prefix-cache/README.md) for new deployments.

## When to Use

- **Low cache hit rate** → Reduce GPU memory, decrease block size
- **OOM errors** → Reduce GPU memory, decrease max length
- **Long contexts needed** → Increase max length, reduce GPU memory, increase SHM
- **High throughput** → Increase GPU memory, standard block size
- **Multi-GPU (TP>2)** → Increase SHM as needed

## Workflow

### 1. Check Current Configuration

```bash
bash skills/llmd-cache-config/scripts/show-current-config.sh ${NAMESPACE}
```

### 2. Update Settings

**Preview first:**
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.90 -b 32 --dry-run
```

**Apply:**
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.90 -b 32
```

**Options:**
- `-g <value>` - GPU memory utilization (0.0-1.0)
- `-b <value>` - Block size in tokens (16-128)
- `-m <value>` - Max model length in tokens
- `-s <value>` - Shared memory size (e.g., 20Gi, 30Gi)

### 3. Verify

```bash
kubectl get pods -n ${NAMESPACE}
kubectl logs -l llm-d.ai/role=decode -n ${NAMESPACE} | grep -E "gpu_memory_utilization|block_size"
```

## Tuning CPU Cache (If Already Enabled)

If your deployment already has CPU offloading enabled via OffloadingConnector or LMCache, you can tune the CPU cache size and InferencePool configuration.

### Adjust CPU Cache Size

**For vLLM OffloadingConnector:**

Edit the deployment to modify `cpu_bytes_to_use` in the `--kv-transfer-config` argument:

```bash
kubectl edit deployment <model-server-name> -n ${NAMESPACE}
```

Find and modify the `cpu_bytes_to_use` value (in bytes):
```yaml
--kv-transfer-config '{"kv_connector":"OffloadingConnector","kv_role":"kv_both","kv_connector_extra_config":{"cpu_bytes_to_use":107374182400}}'
```

Example: Change from 100GB (107374182400) to 150GB (161061273600)

**For LMCache Connector:**

Edit the deployment to modify the `LMCACHE_MAX_LOCAL_CPU_SIZE` environment variable:

```bash
kubectl edit deployment <model-server-name> -n ${NAMESPACE}
```

Find and modify the environment variable (in GB):
```yaml
- name: LMCACHE_MAX_LOCAL_CPU_SIZE
  value: "200.0"  # Change to desired size in GB
```

### Tune InferencePool Prefix Cache Scorers

**What are prefix cache scorers?**
The InferencePool uses scorers to decide which server should handle each request. When CPU offloading is enabled, you configure separate scorers for GPU cache and CPU cache to help route requests to servers that already have relevant cached data.

**Tuning the configuration:**

```bash
helm upgrade llm-d-infpool -n ${NAMESPACE} -f <your-values-file> \
    oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool --version v1.4.0
```

**Key parameters:**

1. **`lruCapacityPerServer`**: Total CPU cache capacity per server (in blocks)
   - Must be manually configured since vLLM doesn't emit CPU block metrics
   - Example: `41000` blocks = ~100GB for Qwen-32B (41,000 blocks × 2.5MB/block)
   - Calculation: 160KB/token × 16 block size = 2.5MB/block
   - Adjust based on your model's block size (check vLLM logs)

2. **Scorer weights**: Balance how the InferencePool prioritizes different factors
   - Default: queue (2.0), kv-cache-util (2.0), gpu-prefix (1.0), cpu-prefix (1.0)
   - CPU cache is a superset of GPU cache (CPU offloading copies GPU entries to CPU)
   - Combined GPU + CPU prefix scorer weight (1.0 + 1.0 = 2.0) balances with other scorers
   - Tune the ratio between GPU and CPU scorers based on your workload

See [`guides/tiered-prefix-cache/cpu/manifests/inferencepool/values.yaml`](../../guides/tiered-prefix-cache/cpu/manifests/inferencepool/values.yaml) for full configuration example.

## Common Scenarios

### Increase Cache Hit Rate
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.88 -b 32
```
Reduces GPU memory (0.95→0.88) for more cache, decreases block size (64→32) for finer matching.

### Support Longer Contexts
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -m 16384 -g 0.85 -s 30Gi
```
Increases max length (8192→16384), reduces GPU memory (0.95→0.85), increases SHM (20Gi→30Gi).

### Maximize Throughput
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.95 -b 64
```
Increases GPU memory (0.90→0.95) for more capacity, standard block size (32→64).

### Fix OOM Errors
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.85
```
Reduces GPU memory (0.95→0.85) to reduce memory pressure.

### Adjust Shared Memory for Multi-GPU
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -s 50Gi
```
Increases SHM (20Gi→50Gi) based on tensor parallelism configuration.

### Manual Edits for Non-Standard Deployments
If script fails, manually edit configuration files:
1. Update `values-modelservice.yaml`: Change `--block-size` and `--gpu-memory-utilization`
2. Update `values-inferencepool.yaml`: Adjust `lruCapacityPerServer`
3. Apply: `cd deployment-dir && helmfile apply -n ${NAMESPACE}`
4. Verify: `kubectl rollout status deployment/<name> -n ${NAMESPACE}`

## Monitoring & Troubleshooting

### Monitoring Commands
```bash
# KV Cache Usage
kubectl logs -l llm-d.ai/role=decode -n ${NAMESPACE} | grep "kv_cache_usage"

# GPU Memory
kubectl exec <pod> -n ${NAMESPACE} -- nvidia-smi

# Cache Hit Rate
kubectl logs -l inferencepool=<pool> -n ${NAMESPACE} | grep "cache_hit_rate"
```

### Common Issues
- **OOM Errors**: Reduce GPU memory (`-g 0.85`), reduce max length, check `nvidia-smi`
- **Low Cache Hit Rate**: Decrease block size (`-b 32`), verify block size matches in gaie-values.yaml
- **SHM Errors**: Increase SHM (`-s 40Gi`), verify with `kubectl exec <pod> -- df -h /dev/shm`
- **Pods Not Restarting**: Force restart `kubectl rollout restart deployment/<name> -n ${NAMESPACE}`

## Safety Checklist

- ✅ Check current config first
- ✅ Use `--dry-run` to preview
- ✅ Automatic backups in `deployments/<name>/backups/`
- ✅ Rolling updates maintain availability
- ✅ Verify settings in pod logs after changes

## Related Resources

### Skills
- **llm-d-deployment**: Initial deployment
- **llmd-scale-workers**: Scale worker replicas

### Guides
- **[Tiered Prefix Cache](../../guides/tiered-prefix-cache/README.md)**: Comprehensive guide on prefix cache offloading strategies
  - **[CPU Offloading](../../guides/tiered-prefix-cache/cpu/README.md)**: Initial setup requires redeployment; tuning can be done on existing deployments
  - **[Storage Offloading](../../guides/tiered-prefix-cache/storage/README.md)**: Requires redeployment
- **[Inference Scheduling](../../guides/inference-scheduling/README.md)**: Prefix-aware request scheduling optimizations

## Scripts Reference

See [scripts/README.md](scripts/README.md) for detailed documentation.