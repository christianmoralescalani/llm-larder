# Model formats & quantization

**Summary.** The same model weights can be stored in several **formats** (file containers) and **precisions** (bytes per weight). Format = how the file is organized and loaded; quantization = how many bits each weight uses. For local inference the practical stack is: **GGUF** (llama.cpp ecosystem, single file, mixed quant) or **safetensors** (Hugging Face / Transformers / vLLM, usually fp16/bf16 or 8-bit). Quantization trades a little quality for a lot of memory and speed — Q4_K_M is the community default because it's nearly lossless for 7B+ models.

## File formats

| Format | Ecosystem | Notes |
|---|---|---|
| **GGUF** | llama.cpp, LM Studio, Ollama | Single file (or sharded), metadata + weights together, supports *mixed* quantization (different tensors at different bits), zero-copy loading. The standard for local/consumer hardware. |
| **safetensors** | Hugging Face, Transformers, vLLM, SGLang | Safe (no executable pickle) tensor container, zero-copy, fast. Usually fp16/bf16 or GPTQ/AWQ int4. The standard for serving and fine-tuning. |
| **PyTorch (.bin/.pth)** | legacy | Pickle-based — can execute code on load, slow, being phased out. |
| **ONNX / TensorRT** | specialized runtimes | Compiled graphs, max GPU perf, painful to produce. |
| **MLX** | Apple Silicon | Native format for Apple's MLX framework. |

**GGUF internals**: a header (metadata: architecture, context length, chat template, tensor info) followed by tensor data. Tensors are stored in blocks with per-block scales (see quantization below). `llama-quantize` produces these; HF hosts thousands of pre-quantized GGUFs (e.g. `Qwen/Qwen3-8B-GGUF`).

## Quantization: the precision ladder

Full precision is fp32 (4 bytes/weight); modern training uses bf16/fp16 (2 bytes). Quantization stores weights in fewer bits:

| Level | Bytes/weight (approx) | What it is |
|---|---|---|
| FP16 / BF16 | 2.0 | "full precision" today; baseline quality |
| Q8_0 | ~1.06 | 8-bit int + per-block scale; ~lossless |
| Q6_K | ~0.82 | 6-bit "K-quants" (super-block layout) |
| Q5_K_M | ~0.72 | 5-bit K-quant, medium mix |
| Q4_K_M | ~0.61 | 4-bit K-quant, medium mix — **the default** |
| Q4_0 | ~0.57 | 4-bit, simple per-block scale |
| Q3_K_M | ~0.45 | 3-bit; quality drops noticeably |
| Q2_K | ~0.32 | 2-bit; only for tight spaces |

"K-quants" (from ggml, verified from source `ggml-common.h`): weights are grouped in 256-element super-blocks, each with a scale + min stored in half precision, and the scales themselves are quantized. The `_S/_M/_L` suffix = mix of Q6/Q5/Q4 blocks: some tensors (attention output, embeddings) stay at higher precision where they're more sensitive. That's why Q4_K_M is ~0.61 B/weight, not a pure 4-bit 0.5.

**Real file sizes** (verified from Hugging Face, 2026-10-27):

| Model (total params) | Q4_K_M | Q5_K_M | Q6_K | Q8_0 |
|---|---|---|---|---|
| Qwen3-8B (8.17B) | 5.03 GB (0.615 B/p) | 5.85 GB | 6.73 GB | 8.71 GB |
| Llama-3.1-8B (8.03B) | ~4.9 GB | ~5.7 GB | ~6.6 GB | ~8.6 GB |

## Other quantization families (not GGUF)

- **GPTQ / AWQ** (int4, safetensors): calibration-based, for vLLM/Transformers GPU serving. AWQ protects "salient" weights; GPTQ uses second-order error compensation.
- **bitsandbytes / LLM.int8()**: on-the-fly 8-bit, no calibration, for fine-tuning + inference in PyTorch.
- **FP8 (E4M3/E5M2)**: hardware-native on Ada/Hopper/Blackwell GPUs; near-fp16 quality at half the memory.
- **MXFP4 / NVFP4**: 4-bit *floating point* micro-block formats (Blackwell); the new frontier for training + inference.

## Choosing a quant (local, consumer hardware)

1. **Fit first**: total params × bytes/weight + KV cache + ~1 GB overhead ≤ your VRAM (see `02-hardware-and-vram/`).
2. **Default: Q4_K_M** for 3B+ models. Q5_K_M or Q6_K if you have headroom and want the last bit of quality. Q8_0 if the model is small enough (≤8B on 24 GB) and you want maximum fidelity.
3. **Below Q4_K_M** (Q3, Q2): only when nothing else fits; expect visible quality loss, worse on small models.
4. **Small models (<3B) tolerate less quantization** — fewer redundant parameters to absorb rounding error.
5. **KV cache quantization** (llama.cpp `--cache-type-k q8_0`) is a separate, free-ish win for long contexts.
6. **Quality check**: run the same 5–10 prompts across Q4_K_M vs Q6_K vs Q8_0; the gap is usually small for 7B+ models and large for 1B models.

## My experiments

- (fill in: e.g. "side-by-side Q4_K_M vs Q8_0 on Qwen3-8B: same prompts, Q4 slightly less consistent on code; tg speed 48 vs 31 tok/s")
