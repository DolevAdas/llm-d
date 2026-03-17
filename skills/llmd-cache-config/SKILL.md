---
name: llmd-cache-config
description: Modify cache memory settings in existing llm-d deployments without full redeployment. Adjust GPU memory utilization, KV cache capacity, shared memory, block size, and context length to optimize performance for different workload patterns. Use when you need to tune cache settings, increase throughput, reduce latency, or support longer contexts.
---

# llm-d Cache Configuration Skill

## Overview

Modify cache memory settings in existing llm-d deployments to optimize performance without requiring full redeployment. Supports adjusting GPU memory utilization, KV cache capacity, shared memory (SHM), block size, and maximum context length.

## When to Use

- Increase KV cache capacity for better prefix cache hit rates
- Adjust GPU memory utilization to balance model size vs cache space
- Modify shared memory for multi-process scenarios
- Change block size for finer-grained prefix matching
- Extend context length for longer prompts/responses
- Optimize cache settings for specific workload patterns

## Prerequisites

- Existing llm-d deployment in Kubernetes/OpenShift
- kubectl or oc CLI with appropriate permissions
- Access to deployment configuration files (ms-values.yaml)

## Workflow

### 1. Check Current Configuration

```bash
bash skills/llmd-cache-config/scripts/show-current-config.sh ${NAMESPACE}
```

Shows current cache settings, resource usage, and deployment info.

### 2. Update Cache Settings

**Preview changes first:**
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.90 -b 32 --dry-run
```

**Apply changes:**
```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.90 -b 32
```

**Options:**
- `-g <value>` - GPU memory utilization (0.0-1.0)
- `-b <value>` - Block size in tokens
- `-m <value>` - Max model length in tokens
- `-s <value>` - Shared memory size (e.g., 20Gi, 30Gi)

## Cache Parameters

### GPU Memory Utilization
**Parameter**: `--gpu-memory-utilization`  
**Default**: 0.95 (95%)  
**Range**: 0.0-1.0  
**Purpose**: Controls fraction of GPU memory for model vs KV cache

Lower values = more KV cache space

### Block Size
**Parameter**: `--block-size`  
**Default**: 64 tokens  
**Range**: 16-128  
**Purpose**: KV cache granularity for prefix matching

Smaller blocks = finer prefix matching, more overhead

### Max Model Length
**Parameter**: `--max-model-len`  
**Default**: Model-specific  
**Purpose**: Maximum context window (prompt + response)

Longer contexts = more memory per request

### Shared Memory (SHM)
**Parameter**: `sizeLimit` in shm volume  
**Default**: 20Gi  
**Purpose**: Inter-process communication for multi-GPU setups

Formula: `10Gi + (5Gi × tensor_parallelism)`

## Common Scenarios

### Increase Cache Hit Rate
**Goal**: Better prefix cache performance

```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.88 -b 32
```

**Changes:**
- GPU memory: 0.95 → 0.88 (more cache space)
- Block size: 64 → 32 (finer granularity)

### Support Longer Contexts
**Goal**: Handle longer prompts/responses

```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -m 16384 -g 0.85 -s 30Gi
```

**Changes:**
- Max length: 8192 → 16384 (double context)
- GPU memory: 0.95 → 0.85 (accommodate longer contexts)
- SHM: 20Gi → 30Gi (larger batches)

### Maximize Throughput
**Goal**: More concurrent requests

```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -g 0.95 -b 64
```

**Changes:**
- GPU memory: 0.90 → 0.95 (maximize model capacity)
- Block size: 32 → 64 (standard granularity)

### High Tensor Parallelism
**Goal**: Support TP=8 for large models

```bash
bash skills/llmd-cache-config/scripts/update-cache-config.sh \
  -n ${NAMESPACE} -s 50Gi
```

**Changes:**
- SHM: 20Gi → 50Gi (10Gi + 5Gi × 8)

## Monitoring

### Key Metrics

**KV Cache Utilization:**
```bash
kubectl logs -l llm-d.ai/role=decode -n ${NAMESPACE} | grep "kv_cache_usage"
```
Target: 60-80% for optimal performance

**GPU Memory:**
```bash
kubectl exec <pod> -n ${NAMESPACE} -- nvidia-smi
```
Target: 85-95% utilization

**Cache Hit Rate** (if using prefix caching):
```bash
kubectl logs -l inferencepool=<pool> -n ${NAMESPACE} | grep "cache_hit_rate"
```
Target: >50% for workloads with repeated prefixes

### Resource Usage
```bash
kubectl top pods -n ${NAMESPACE}
```

## Troubleshooting

### OOM (Out of Memory) Errors
**Symptoms**: Pods crash with OOMKilled

**Solutions:**
1. Reduce GPU memory utilization: `-g 0.90` → `-g 0.85`
2. Reduce max model length if set too high
3. Check actual GPU memory: `kubectl exec <pod> -- nvidia-smi`

### Low Cache Hit Rate
**Symptoms**: Poor performance despite prefix caching

**Solutions:**
1. Decrease block size: `-b 64` → `-b 32`
2. Verify block size matches in gaie-values.yaml
3. Check if workload has repeated prefixes

### Shared Memory Errors
**Symptoms**: Errors about `/dev/shm` being full

**Solutions:**
1. Increase SHM size: `-s 30Gi` → `-s 40Gi`
2. Verify SHM is mounted: `kubectl exec <pod> -- df -h /dev/shm`

### Pods Not Restarting
**Symptoms**: Old configuration still in use

**Solutions:**
1. Force restart: `kubectl rollout restart deployment/<name> -n ${NAMESPACE}`
2. Check Helm release: `helm list -n ${NAMESPACE}`

## Performance Tuning Guidelines

### Memory Allocation Strategy

**Total GPU Memory = Model Weights + KV Cache + Overhead**

**Recommended GPU Memory Utilization:**
- Small models (<20B): 0.95
- Medium models (20-70B): 0.90-0.93
- Large models (70B+): 0.85-0.90
- MoE models: 0.90-0.92

### Block Size Selection

**Guidelines:**
- Short prefixes (<100 tokens): 16-32
- Medium prefixes (100-500 tokens): 32-64
- Long prefixes (>500 tokens): 64-128
- Mixed workload: 32-64

### Shared Memory Sizing

**Formula**: `SHM = 10Gi + (5Gi × tensor_parallelism)`

**Examples:**
- TP=1: 15Gi
- TP=2: 20Gi
- TP=4: 30Gi
- TP=8: 50Gi

## Safety Checklist

Before modifying cache settings:
- ✅ Check current configuration with show-current-config.sh
- ✅ Use --dry-run to preview changes
- ✅ Understand workload characteristics
- ✅ Have rollback plan (backups created automatically)
- ✅ Monitor during rollout
- ✅ Verify settings in pod logs
- ✅ Test inference after changes

## Scripts Reference

See [scripts/README.md](scripts/README.md) for detailed script documentation.

**Available scripts:**
- `show-current-config.sh` - Display current cache configuration
- `update-cache-config.sh` - Update cache settings with rolling update

## Related Skills

- **llm-d-deployment**: Initial deployment of llm-d stack
- **llmd-scale-workers**: Scale replicas for throughput

## Additional Resources

- [vLLM Memory Management](https://docs.vllm.ai/en/latest/models/performance.html)
- [Prefix Caching Guide](https://docs.vllm.ai/en/latest/automatic_prefix_caching/apc.html)
- [llm-d Architecture](https://llm-d.ai/docs/architecture)