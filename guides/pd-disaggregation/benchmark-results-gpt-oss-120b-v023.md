# P/D Disaggregation Benchmark Results — llm-d v0.8.0 / vLLM v0.23.0

**Model**: openai/gpt-oss-120b  
**llm-d version**: v0.8.0 (main)  
**vLLM version**: v0.23.0  
**Hardware**: 16× H100 GPUs (on-premise bare-metal, UCX RoCE via NIXL)  
**Workload**: ISL=5000 / OSL=250, constant rate 45 req/s × 120s, 45 workers, max_concurrency=100

> [!NOTE]
> The original benchmarking report in this guide was collected on 16× H200 GPUs with InfiniBand on CKS.
> These results were collected on a different cluster (16× H100, UCX RoCE) running a newer vLLM version (v0.23.0).
> Absolute numbers are not directly comparable across hardware; the relative P/D vs aggregated improvement within each run is the meaningful signal.

---

## Configurations

| Run | Layout | GPUs | P/D | Description |
| --- | ------ | ---- | --- | ----------- |
| **A1** | 16× TP=1 | 16 | ❌ | Aggregated baseline — guide pd-disaggregation layout |
| **A2** | 8× TP=2 | 16 | ❌ | Aggregated baseline — 8-pod TP=2 layout |
| **B** | 4P(TP=2) + 2D(TP=4) | 8+8=16 | ✅ | P/D disaggregation — UCX GDR (`rc,cuda,sm` prefill / `cuda,rc,sm` decode) |

All runs:

- `--gpu-memory-utilization=0.90`, `--max-model-len=8192`, `--block-size=128`
- `NixlConnector`, `kv_role=kv_both`, `--no-disable-hybrid-kv-cache-manager` (P/D only)
- `UCX_MAX_RMA_RAILS=1`, `UCX_TLS=rc,cuda,sm` prefill / `cuda,rc,sm` decode (P/D only)
- `rdma/roce_gdr: "1"` in pod resource limits/requests for RDMA kernel-bypass (P/D only)

---

## Comparing llm-d P/D Disaggregation to a k8s Service

For this workload (ISL=5000 / OSL=250, 45 req/s), llm-d P/D disaggregation improved mean E2E latency by **68%** vs the 16×TP=1 baseline (A1) and **82%** vs the 8×TP=2 baseline (A2), with ITL improvements of 51% and 92% respectively.

### vs A1 (16×TP=1 — guide default layout)

A1 matches the guide's default aggregated layout (16 replicas of TP=1 behind a k8s Service).

| Metric | Aggregated (16×TP=1) | llm-d P/D | Δ% |
| :----- | :------ | :------ | :----- |
| **E2E Latency (Mean)** | **17.1s** | **5.5s** | **-68%** |
| **E2E Latency (P90)** | **22.2s** | **7.3s** | **-67%** |
| ITL (Mean) | 18.8ms | 9.2ms | -51% |
| ITL (P90) | 12.6ms | 13.8ms | +10% |
| TTFT (Mean) | 12,354ms | 3,154ms | -74% |
| TTFT (P90) | 17,712ms | 4,974ms | -72% |

### vs A2 (8×TP=2 — same TP as P/D prefill pods)

A2 uses the same TP=2 as the P/D prefill pods, providing a like-for-like comparison of the decode path.

| Metric | Aggregated (8×TP=2) | llm-d P/D | Δ% |
| :----- | :------ | :------ | :----- |
| **E2E Latency (Mean)** | **30.4s** | **5.5s** | **-82%** |
| **E2E Latency (P90)** | **49.7s** | **7.3s** | **-85%** |
| ITL (Mean) | 116ms | 9.2ms | -92% |
| ITL (P90) | 141ms | 13.8ms | -90% |
| TTFT (Mean) | 1,322ms | 3,154ms | +139% |
| TTFT (P90) | 2,338ms | 4,974ms | +113% |

> [!NOTE]
> In the A2 comparison, TTFT is elevated in the disaggregated setup because fewer
> GPU resources are allocated to prefill processing (4 prefill pods vs 8 pods in A2).
> At ISL=5000 / OSL=250, the decode phase dominates total E2E latency: in A2, 95%
> of E2E time (28.9s of 30.4s) is decode. P/D collapses the decode term from 28.9s
> to 2.3s, making the TTFT increase irrelevant to overall latency. The OSL break-even
> for P/D to win on E2E mean is only ~18 output tokens.

---

## Detailed Results

### Run A1: 16× TP=1 Aggregated Baseline

**Status**: ✅ 5400/5400 = 100% success

| Metric | Mean | p50 | p90 | p99 |
| ------ | ---- | --- | --- | --- |
| **TTFT** | 12,354 ms | 12,464 ms | 17,712 ms | 21,108 ms |
| **TPOT** | 18.8 ms | 17.1 ms | 23.6 ms | 39.9 ms |
| **ITL** | 18.8 ms | 11.9 ms | 12.6 ms | 141 ms |
| **E2E** | 17,083 ms | 17,008 ms | 22,216 ms | 25,741 ms |

Output tokens/s: 10,636 · Req/s: 38.27 (target: 45.0)

---

### Run A2: 8× TP=2 Aggregated Baseline

**Status**: ✅ 5396/5400 = 99.9% success

| Metric | Mean | p50 | p90 | p99 |
| ------ | ---- | --- | --- | --- |
| **TTFT** | 1,322 ms | 1,178 ms | 2,338 ms | 3,731 ms |
| **TPOT** | 116.2 ms | 111.4 ms | 191.0 ms | 214.0 ms |
| **ITL** | 116.2 ms | 29.6 ms | 140.6 ms | 1,070 ms |
| **E2E** | 30,393 ms | 29,080 ms | 49,690 ms | 55,352 ms |

Output tokens/s: 11,429 · Req/s: 42.44 (target: 45.0)

---

### Run B: 4P(TP=2) + 2D(TP=4) P/D Disaggregation (with RDMA)

**Status**: ✅ 5400/5400 = 100% success

| Metric | Mean | p50 | p90 | p99 |
| ------ | ---- | --- | --- | --- |
| **TTFT** | 3,154 ms | 2,798 ms | 4,974 ms | 10,071 ms |
| **TPOT** | 9.2 ms | 9.2 ms | 10.5 ms | 11.5 ms |
| **ITL** | 9.2 ms | 8.9 ms | 13.8 ms | 22.3 ms |
| **E2E** | 5,468 ms | 5,194 ms | 7,299 ms | 12,483 ms |

Output tokens/s: 12,171 · Req/s: 43.24 (target: 45.0)

TPOT is clean and unimodal (p50=9.2ms, p90=10.5ms, p99=11.5ms). Both decode pods use kernel-bypass UCX transport (RDMA) for KV cache transfers regardless of node co-location.
