# Benchmarks

## Single-Spark vs Dual-Spark Comparison

These benchmarks were collected during the investigation that led to this single-stack configuration.

### Decode Speed (tok/s)

| Configuration | Engine | MTP | Tok/s | Hardware |
|--------------|--------|-----|-------|----------|
| Dual-Spark (Ray, TP=2) | vLLM | No | ~28 | 2× DGX Spark |
| Single-Spark (TP=1) | vLLM | No | 29.5 | 1× DGX Spark |
| Single-Spark (NVFP4) | Atlas | No | 31.1 | 1× DGX Spark |
| Dual-Spark (EP=2, RoCE) | Atlas | Yes | 42.2 | 2× DGX Spark |
| **Single-Spark (hybrid INT4+FP8)** | **vLLM** | **Yes (K=3)** | **41–47** | **1× DGX Spark** |

### Key Finding

Single-Spark with MTP **matches or exceeds** dual-Spark performance because:
- Ray inter-node overhead (~2–5% per token) eats the second Spark's contribution
- MTP (Multi-Token Prediction) only works reliably on single-node
- MTP gives 40–50% speedup at concurrency=1

### tool-eval-bench Results

Using Nic's `tool-eval-bench` with the hybrid INT4+FP8 configuration:

| Mode | Score | Median Turn | Tasks |
|------|-------|-------------|-------|
| Sequential | 91/100 | 2.0s | 15 (short) |
| Parallel-4 | 88/100 | 5.9s | 15 (short) |
| Parallel-15 | Pass | 90s total | 15 (short) |

100% pass rate on the 15 short test tasks. Some partial scores on the full 69-task suite (consistent with other models).

### TTFT (Time to First Token)

| Prompt Size | TTFT |
|------------|------|
| ~19 tokens | 1.0s |
| ~3,500 tokens | 6.8s |
| ~7,000 tokens (typical agent) | ~10–13s |

TTFT scales roughly linearly with prompt size. For agent use with large system prompts, this is the primary latency bottleneck — not decode speed.

### Concurrent Load Impact

With Nic's OOM-fix configuration (`--max-num-batched-tokens 16384`, `--max-num-seqs 256`, `--enable-chunked-prefill`):

- **15 parallel tasks:** Completed in 90 seconds, stable, no OOM
- **MTP benefit under load:** +10–20% (lower than the +40–50% at concurrency=1)
- **Prefix caching:** Helps when multiple requests share system prompts

### Resource Coexistence

With all services running (vLLM + Hermes + Honcho + Uptime Kuma):

| Metric | Value |
|--------|-------|
| Total RAM usage | ~69 GB (GPU) + ~1 GB (CPU services) |
| GPU utilisation (idle) | ~0% (model loaded but waiting) |
| GPU utilisation (generating) | ~85% (as configured) |
| CPU usage (idle) | <5% |
| Inference speed impact from co-running services | **None measurable** |

The agent stack has no observable impact on inference performance.

## Methodology Notes

- Tok/s measurements use wall-clock time from request start to response completion
- Benchmarks run on DGX Spark with 128 GB unified memory, Grace CPU, Blackwell GPU
- Room temperature approximately 22°C (thermal throttling not observed)
- All benchmarks with thinking mode disabled
