# vLLM Configuration Reference

## Docker Run Command

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

## Flag Reference

| Flag | Value | Why |
|------|-------|-----|
| `--gpu-memory-utilization` | `0.85` | **Critical.** 0.90 causes OOM crashes under concurrent load. |
| `--max-num-batched-tokens` | `16384` | **Critical.** Default (32768) exhausts memory with MTP enabled. |
| `--max-num-seqs` | `256` | **Critical.** Caps concurrent sequences to prevent memory exhaustion. |
| `--enable-prefix-caching` | — | Reuses KV cache for shared prefixes (system prompts, tools). |
| `--enable-chunked-prefill` | — | Chunks large prompts instead of processing in one allocation. |
| `--speculative-config` | `{"method":"mtp","num_speculative_tokens":2}` | MTP-2 (K=3). +40–50% at concurrency=1, +10–20% under load. |
| `--attention-backend` | `FLASHINFER` | Faster than FlashAttention on Blackwell / GB10 architecture. |
| `--load-format` | `fastsafetensors` | Required for hybrid INT4+FP8 weights. |
| `--tool-call-parser` | `qwen3_coder` | Qwen3-specific tool-call parser for function calling. |
| `--chat-template` | `unsloth.jinja` | Suppresses thinking/reasoning tokens (see [thinking-modes.md](thinking-modes.md)). |
| `--generation-config` | `auto` | Loads model's default sampling parameters. |

## The OOM Fix

### Root Cause

```
--gpu-memory-utilization 0.90
+ no sequence limit (default: unlimited)
+ default max-num-batched-tokens (32768)
= unified memory exhaustion under concurrent load
```

### Solution

```bash
--gpu-memory-utilization 0.85        # 5% headroom
--max-num-batched-tokens 16384       # Half the default
--max-num-seqs 256                   # Explicit cap
--enable-prefix-caching              # Reuse shared KV cache
--enable-chunked-prefill             # Don't prefill huge prompts in one go
```

MTP was initially suspected but is NOT the cause. It was removed during diagnosis, then restored once the guardrails were in place.

### What NOT to Set

| Flag / Value | Why It Fails |
|-------------|-------------|
| `--gpu-memory-utilization 0.90` | OOM during CUDA graph capture with MTP |
| `num_speculative_tokens: 3` (K=4) | OOM — needs more GPU memory than available |
| `--max-num-batched-tokens 32768` | Default, too large with MTP enabled |
| `--swap-space` | Not supported by this vLLM build |
| `--disable-hybrid-kv-cache-manager` | Not supported by this vLLM build |
| `--reasoning-parser qwen3` | Causes `content: null` in API responses |

## Memory Budget (Approximate)

| Component | Size |
|-----------|------|
| Model weights (hybrid INT4+FP8) | ~67 GB |
| KV cache (at 0.85 util) | ~40 GB |
| CUDA graphs + compilation | ~1.3 GB |
| MTP speculative heads | ~2 GB |
| **Total GPU** | **~110 GB of 128 GB** |
| System/CPU overhead | ~2–3 GB |
| **Available for agent stack** | **~15–18 GB** |

## Startup Sequence

1. **Model loading:** ~65 seconds (fastsafetensors format)
2. **torch.compile:** ~50 seconds (cached after first run)
3. **CUDA graph capture:** ~13 seconds
4. **FlashInfer autotune:** ~30 seconds
5. **Multi-modal warmup:** ~10 seconds
6. **Total cold start:** ~200 seconds (~3.5 minutes)
7. **Warm restart (cached):** ~120 seconds (~2 minutes)

## vLLM Version

Built on vLLM `0.19.2rc1` with custom patches:
- INT4+FP8 hybrid weight loading
- MTP head injection
- FlashInfer integration for GB10
- INT8 LM Head optimisation

See [hybrid-checkpoint.md](hybrid-checkpoint.md) for the Docker image build process.
