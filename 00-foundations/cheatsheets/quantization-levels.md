# Cheatsheet — Quantization levels (GGUF)

**TL;DR.** File size ≈ `params × bytes/weight`. For local models, **Q4_K_M is the default** (~0.61 B/weight, nearly lossless for 3B+ models); go Q6_K/Q8_0 for max quality if VRAM allows, Q3_K_M or lower only when nothing else fits. Decode speed scales ~linearly with bytes/weight (bandwidth-bound).

## Bytes per weight (GGUF, from ggml source + verified file sizes)

| Quant | B/weight (pure block) | B/weight (real, measured) | Quality vs Q8_0 |
|---|---|---|---|
| FP16/BF16 | 2.0 | ~2.0 | baseline |
| Q8_0 | 1.0625 | **1.066** (Qwen3-8B: 8.71 GB) | ~lossless |
| Q6_K | 0.84375 | **0.823** (6.73 GB) | ~lossless |
| Q5_K_M | ~0.72 | **0.716** (5.85 GB) | negligible loss |
| Q5_0 | 0.703125 | **0.700** (5.72 GB) | negligible loss |
| Q4_K_M | 0.5625 (pure Q4_K) | **0.615** (5.03 GB) | very small loss |
| Q4_0 | 0.5625 | ~0.57 | small loss |
| Q3_K_M | ~0.45 | ~0.45–0.48 | noticeable on hard tasks |
| Q2_K | ~0.33 | ~0.32 | significant loss |

Measured on `Qwen/Qwen3-8B-GGUF` (8.17B params) and `Qwen/Qwen2.5-7B-Instruct-GGUF` (7.62B) via Hugging Face file sizes, 2026-10-27.

## Why Q4_K_M ≠ 0.5 B/weight

K-quants use 256-element super-blocks with per-block scales + mins (half precision), and the `_M` mix keeps sensitive tensors (attention output, some embeddings) at Q5/Q6. So the *effective* density is ~0.61, not 0.5. The "0.57" you'll see quoted is the pure Q4_K block density.

## Size formula (quick VRAM check)

```text
weights GB ≈ params(B) × bytes/weight ÷ 1.0
+ KV cache (see kv-cache-formula.md)
+ ~0.5–1 GB runtime overhead
```

| Model | Q4_K_M weights | Q8_0 weights |
|---|---|---|
| 4B | ~2.5 GB | ~4.3 GB |
| 8B | ~5.0 GB | ~8.5 GB |
| 14B | ~8.6 GB | ~14.9 GB |
| 32B | ~19.7 GB | ~34.1 GB |
| 70B | ~43.1 GB | ~74.7 GB |

## Quality rules of thumb

- 7B+ models: Q4_K_M ≈ Q6_K ≈ Q8_0 for most tasks.
- 1–3B models: prefer Q6_K/Q8_0 — less redundancy to absorb rounding error.
- Below Q4_K_M: expect visible degradation (repetition, weaker reasoning), worse on small models.
- KV cache quant is a separate axis: `--cache-type-k q8_0 --cache-type-v q8_0` saves ~half the cache VRAM for ~no quality loss.

## Non-GGUF quant families (GPU serving)

| Family | Bits | Ecosystem | Notes |
|---|---|---|---|
| GPTQ | 4 (sometimes 3/8) | vLLM, Transformers | calibration-based |
| AWQ | 4 | vLLM, Transformers | protects salient weights, calibration-based |
| bitsandbytes | 8 | Transformers | on-the-fly, no calibration |
| FP8 (E4M3) | 8 | vLLM, SGLang, native on Ada/Hopper/Blackwell | near-fp16 quality |
| NVFP4 / MXFP4 | 4 | Blackwell | 4-bit float micro-blocks |

## Verified on

- ggml source (`ggml-common.h` block definitions, QK_K=256, K_SCALE_SIZE=12), fetched 2026-10-27.
- File sizes: Hugging Face API for `Qwen/Qwen3-8B-GGUF` and `Qwen/Qwen2.5-7B-Instruct-GGUF`, 2026-10-27.
