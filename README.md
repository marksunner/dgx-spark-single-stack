# DGX Spark Single-Stack AI Agent

**A complete autonomous AI agent running on a single NVIDIA DGX Spark.**

One box. One GPU. Full-stack AI: inference, agent runtime, conversation memory, and monitoring — all on a single 128GB DGX Spark.

## What This Is

A documented, reproducible configuration for running:

| Component | Role | Resource |
|-----------|------|----------|
| **Qwen 3.5 122B** | LLM inference (hybrid INT4+FP8, MTP speculative decoding) | GPU (~67 GB) |
| **Hermes Agent** | AI agent runtime with tool use, Discord/chat integration | CPU (~300 MB) |
| **Honcho** | Conversation memory (PostgreSQL + Redis) | CPU (~500 MB) |
| **Uptime Kuma** | Health monitoring dashboard | CPU (~100 MB) |

The key insight: **GPU and CPU workloads don't compete.** vLLM claims GPU memory exclusively via `--gpu-memory-utilization`, while the agent stack runs entirely on CPU/system RAM. They coexist comfortably.

## Performance

| Metric | Value |
|--------|-------|
| Decode speed | **41–47 tok/s** (single user) |
| MTP acceptance | +40–50% at concurrency=1 |
| Concurrent stability | 15 parallel tasks, 100% pass rate |
| Sequential benchmark | 91/100 score, 2.0s median turn |
| Parallel-4 benchmark | 88/100 score, 5.9s median turn |
| Model load time | ~65 seconds |
| Full init (compile + warmup) | ~200 seconds |

## Quick Start

### 1. Build the Hybrid Checkpoint

See [`docs/hybrid-checkpoint.md`](docs/hybrid-checkpoint.md) for the full process.

The hybrid INT4+FP8 checkpoint merges Intel AutoRound INT4 weights with Qwen FP8 weights, yielding better throughput than either format alone.

### 2. Launch vLLM

```bash
docker run -d --name vllm-qwen35 \
    --gpus all --net=host --ipc=host \
    -v /path/to/models:/models \
    vllm-qwen35-v2 \
    serve /models/qwen35-122b-hybrid-int4fp8 \
    --served-model-name qwen3.5-122b \
    --max-model-len 131072 \
    --gpu-memory-utilization 0.85 \
    --port 8000 --host 0.0.0.0 \
    --load-format fastsafetensors \
    --attention-backend FLASHINFER \
    --enable-chunked-prefill \
    --max-num-batched-tokens 16384 \
    --max-num-seqs 256 \
    --enable-prefix-caching \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --generation-config auto \
    --chat-template /models/qwen35-122b-hybrid-int4fp8/unsloth.jinja \
    --override-generation-config '{"temperature": 0.3, "top_p": 0.95, "top_k": 20, "presence_penalty": 0.0, "repetition_penalty": 1.0}' \
    --speculative-config '{"method":"mtp","num_speculative_tokens":2}'
```

### 3. Install Hermes Agent

See [`docs/hermes-setup.md`](docs/hermes-setup.md).

### 4. Install Honcho (Optional)

See [`docs/honcho-setup.md`](docs/honcho-setup.md).

## Documentation

- [`docs/vllm-config.md`](docs/vllm-config.md) — vLLM flag reference and OOM prevention
- [`docs/hybrid-checkpoint.md`](docs/hybrid-checkpoint.md) — Building the hybrid INT4+FP8 checkpoint
- [`docs/hermes-setup.md`](docs/hermes-setup.md) — Agent runtime installation
- [`docs/honcho-setup.md`](docs/honcho-setup.md) — Conversation memory setup
- [`docs/thinking-modes.md`](docs/thinking-modes.md) — Controlling thinking/reasoning tokens
- [`docs/troubleshooting.md`](docs/troubleshooting.md) — Common issues and fixes
- [`docs/benchmarks.md`](docs/benchmarks.md) — Performance testing methodology

## Credits

This configuration builds on the work of several people:

- **[albond](https://github.com/albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4)** — Hybrid checkpoint recipe, MTP configuration, FlashInfer integration, OOM diagnosis, and `tool-eval-bench`. The breakthrough that made single-Spark 50+ tok/s possible.
- **[eugr](https://github.com/eugr)** — Base DGX Spark infrastructure tooling.
- **Atlas Inference Engine team** — Dual-Spark EP=2 benchmarks that helped validate the single-vs-dual comparison.
- **Qwen team** — The model itself, and the MTP architecture.

## Background

### Why Single Spark?

We previously ran Qwen 3.5 122B across **two** DGX Sparks using Ray distributed inference. The assumption was that more hardware = more speed.

**Wrong.**

| Configuration | Tok/s | Hardware |
|--------------|-------|----------|
| Dual-Spark (Ray, no MTP) | ~28 | 2× DGX Spark |
| Single-Spark (vLLM, no MTP) | ~29.5 | 1× DGX Spark |
| Single-Spark (hybrid INT4+FP8, MTP K=3) | **41–47** | 1× DGX Spark |

Ray's inter-node communication overhead ate any benefit from the second Spark. Meanwhile, MTP (Multi-Token Prediction) — which only works reliably on single-node — gave a 40–50% speedup. The result: **one Spark is faster than two.**

This frees up hardware for other experiments, additional agents, or simply reduces power consumption.

## License

This documentation is shared freely. The referenced tools (vLLM, Hermes, Honcho, Qwen) have their own licenses.
