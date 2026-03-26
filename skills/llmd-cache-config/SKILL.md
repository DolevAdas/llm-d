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

## When to Use

- **Low cache hit rate** → Reduce GPU memory, decrease block size
- **OOM errors** → Reduce GPU memory, decrease max length
- **Long contexts needed** → Increase max length, reduce GPU memory, increase SHM
- **High throughput** → Increase GPU memory, standard block size
- **Multi-GPU (TP>2)** → Increase SHM based on formula

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

## Cache Parameters

| Parameter | Default | Range | Purpose |
|-----------|---------|-------|---------|
| **GPU Memory Utilization** (`-g`) | 0.95 | 0.0-1.0 | Model vs KV cache allocation. Lower = more cache |
| **Block Size** (`-b`) | 64 | 16-128 | Prefix matching granularity. Smaller = finer matching |
| **Max Model Length** (`-m`) | Model-specific | - | Context window size. Higher = more memory per request |
| **Shared Memory** (`-s`) | 20Gi | 10Gi-50Gi | IPC for multi-GPU. Formula: `10Gi + (5Gi × TP)` |

**Note:** When changing block size, also update InferencePool `lruCapacityPerServer` parameter:
- Formula: `(GPU_blocks × old_block_size) / new_block_size`
- Example: 9271 blocks @ 128 → 37084 blocks @ 32

## Common Scenarios

### Increase Cache Hit Rate
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.88 -b 32
```
Changes: GPU 0.95→0.88 (more cache), block 64→32 (finer matching)

**Manual Steps for Non-Standard Deployments:**
If script fails to find `ms-values.yaml`, manually edit:
1. Update `values-modelservice.yaml`: Change `--block-size` and `--gpu-memory-utilization` in both decode and prefill sections
2. Update `values-inferencepool.yaml`: Adjust `lruCapacityPerServer` using formula above
3. Apply: `cd deployment-dir && helmfile apply -n ${NAMESPACE}`
4. Verify: `kubectl rollout status deployment/<name> -n ${NAMESPACE}`

### Support Longer Contexts
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -m 16384 -g 0.85 -s 30Gi
```
Changes: Max length 8192→16384, GPU 0.95→0.85, SHM 20Gi→30Gi

### Maximize Throughput
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.95 -b 64
```
Changes: GPU 0.90→0.95 (more capacity), block 32→64 (standard)

### Fix OOM Errors
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.85
```
Changes: GPU 0.95→0.85 (reduce memory pressure)

### High Tensor Parallelism (TP=8)
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -s 50Gi
```
Changes: SHM 20Gi→50Gi (10Gi + 5Gi × 8)

## Tuning Guidelines

**GPU Memory Utilization by Model Size:**
- Small (<20B): 0.95
- Medium (20-70B): 0.90-0.93
- Large (70B+): 0.85-0.90
- MoE: 0.90-0.92

**Block Size by Prefix Length:**
- Short (<100 tokens): 16-32
- Medium (100-500 tokens): 32-64
- Long (>500 tokens): 64-128
- Mixed: 32-64

**Shared Memory by Tensor Parallelism:**
- TP=1: 15Gi
- TP=2: 20Gi
- TP=4: 30Gi
- TP=8: 50Gi

## Troubleshooting

**OOM Errors**: Reduce GPU memory (`-g 0.85`), reduce max length, check `nvidia-smi`

**Low Cache Hit Rate**: Decrease block size (`-b 32`), verify block size matches in gaie-values.yaml

**SHM Errors**: Increase SHM (`-s 40Gi`), verify with `kubectl exec <pod> -- df -h /dev/shm`

**Pods Not Restarting**: Force restart `kubectl rollout restart deployment/<name> -n ${NAMESPACE}`

## Monitoring

**KV Cache Usage:**
```bash
kubectl logs -l llm-d.ai/role=decode -n ${NAMESPACE} | grep "kv_cache_usage"
```
Target: 60-80%

**GPU Memory:**
```bash
kubectl exec <pod> -n ${NAMESPACE} -- nvidia-smi
```
Target: 85-95%

**Cache Hit Rate:**
```bash
kubectl logs -l inferencepool=<pool> -n ${NAMESPACE} | grep "cache_hit_rate"
```
Target: >50% for prefix caching workloads

## Safety Checklist

- ✅ Check current config first
- ✅ Use `--dry-run` to preview
- ✅ Automatic backups in `deployments/<name>/backups/`
- ✅ Rolling updates maintain availability
- ✅ Verify settings in pod logs after changes

## Related Skills

- **llm-d-deployment**: Initial deployment
- **llmd-scale-workers**: Scale worker replicas

## Scripts Reference

See [scripts/README.md](scripts/README.md) for detailed documentation.