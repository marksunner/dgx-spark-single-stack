# Building the Hybrid INT4+FP8 Checkpoint

## Overview

The hybrid checkpoint merges two quantisation formats:
- **INT4 weights** from [Intel/Qwen3.5-122B-A10B-int4-AutoRound](https://huggingface.co/Intel/Qwen3.5-122B-A10B-int4-AutoRound) (~69 GB)
- **FP8 weights** from [Qwen/Qwen3.5-122B-A10B-FP8](https://huggingface.co/Qwen/Qwen3.5-122B-A10B-FP8) (selected shards)

The result is a ~71 GB checkpoint that combines the memory efficiency of INT4 with the accuracy of FP8 for specific layers, plus injected MTP (Multi-Token Prediction) heads.

## Why Hybrid?

| Format | Size | Tok/s (single Spark) | Notes |
|--------|------|---------------------|-------|
| INT4 AutoRound | ~69 GB | ~29.5 | Good compression, no MTP |
| NVFP4 | ~76 GB | ~31 | Atlas-compatible, no MTP headroom |
| **Hybrid INT4+FP8** | **~71 GB** | **41–47** | MTP works, FlashInfer optimised |

The hybrid format leaves enough GPU headroom for MTP speculative decoding, which is the primary speed multiplier.

## Build Process

### Prerequisites

The build uses [albond's DGX Spark recipe](https://github.com/albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4), which provides:
- `install.sh` — Automated 7-step build script
- `build-hybrid-checkpoint.py` — Merges INT4 + FP8 weights
- Docker image build with custom vLLM patches

### Steps

1. **Clone the build repo:**
   ```bash
   git clone https://github.com/albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4.git
   cd DGX_Spark_Qwen3.5-122B-A10B-AR-INT4
   ```

2. **Run the install script:**
   ```bash
   ./install.sh
   ```
   This performs 7 steps:
   - Install prerequisites
   - Create Python virtual environment
   - Download INT4 model from HuggingFace (~75 GB, ~22 minutes)
   - Download FP8 shards (3 files)
   - Build hybrid checkpoint (merges INT4 + FP8, ~30 minutes)
   - Inject MTP heads
   - Build Docker image

3. **Alternative: Quick build using existing vLLM image**

   If you already have a working vLLM Docker image, you can skip the full build:
   ```bash
   # Build hybrid checkpoint manually
   python build-hybrid-checkpoint.py \
       --int4-path /path/to/Qwen3.5-122B-A10B-int4-AutoRound \
       --fp8-path /path/to/Qwen3.5-122B-A10B-FP8 \
       --output-path /path/to/qwen35-122b-hybrid-int4fp8

   # Use existing vLLM image as base (builds in seconds!)
   docker build -t vllm-qwen35-v2 -f Dockerfile.quick .
   ```
   This "Option C" approach builds in under a second vs 30–60 minutes for a full image build.

### Build Times

| Step | Duration |
|------|----------|
| Model download (INT4) | ~22 minutes (75 GB) |
| FP8 shard download | ~10 minutes (3 shards) |
| Hybrid checkpoint merge | ~30 minutes (14 safetensor shards) |
| Docker image (full build) | 30–60 minutes |
| Docker image (quick, existing base) | < 1 second |
| **Total (full)** | ~90 minutes |
| **Total (quick path)** | ~60 minutes |

### Disk Space

| Item | Size |
|------|------|
| INT4 AutoRound model | ~69 GB |
| FP8 shards (temporary) | ~15 GB |
| Hybrid checkpoint (output) | ~71 GB |
| Docker image | ~24 GB (full) or ~0.1 GB delta (quick) |
| **Total working space needed** | ~180 GB |

You can delete the FP8 shards and optionally the INT4 model after the hybrid checkpoint is built.

## Distributing to Other Sparks

If you have multiple DGX Sparks connected via QSFP:

```bash
# From the source Spark (where the checkpoint lives)
rsync -avP --progress /path/to/qwen35-122b-hybrid-int4fp8/ \
    user@<target-spark-qsfp-ip>:/path/to/models/qwen35-122b-hybrid-int4fp8/
```

Over 200Gbps QSFP: ~72 GB transfers in approximately 2 minutes.
Over Ethernet: ~72 GB transfers in approximately 60 minutes.

The Docker image can also be transferred:
```bash
docker save vllm-qwen35-v2 | ssh user@<target> docker load
```
