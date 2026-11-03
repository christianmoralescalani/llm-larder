# Quantization math: bytes per weight, size, quality

**Summary.** Quantization stores each weight with fewer bits, shrinking the model file and the VRAM it needs. The practical map: **FP16 = 2.0 B/param, Q8_0 ≈ 1.06, Q6_K ≈ 0.84, Q4_K_M ≈ 0.61 (the default), Q3_K_M ≈ 0.49, Q2_K ≈ 0.38**. Quality degrades smoothly from Q8_0 down to Q3_K_M; below Q3, dense models start to noticeably suffer. K-quants (Q4_K_M etc.) beat naive 4-bit because they keep high-precision scales per 256-weight super-block, and IQ/UD quants add an *importance matrix* to spend bits where the model cares.

## Bytes per weight (GGUF, llama.cpp)

| Quant | bits/weight | B/param | Quality | When to use |
|---|---|---|---|---|
| F16 / BF16 | 16 | 2.0 | reference | testing, small models on big GPUs |
| Q8_0 | 8.5 | 1.0625 | ~lossless | max quality that still fits; ≤8B on 24 GB |
| Q6_K | 6.56 | 0.84375 | near-lossless | 14–32B with headroom |
| Q5_K_M | ~5.7 | ~0.72 | tiny degradation | one step above default |
| **Q4_K_M** | ~4.85 | **~0.61** | small degradation | **the default for 3B+ models** |
| Q4_0 | 4.5 | 0.5625 | small degradation | older/simple quants |
| IQ4_XS | ~4.25 | ~0.53 | ≈ Q4_K_M (imatrix) | tight spaces |
| Q3_K_M | ~4.1 | ~0.49 | noticeable degradation | only when nothing else fits |
| IQ3_XXS | ~3.06 | ~0.38 | clear degradation | 70B on 48 GB, MoE |
| Q2_K | ~3.4 | ~0.38 | heavy degradation | last resort |
| IQ2_XXS | ~2.06 | ~0.26 | severe | 405B-class on 128 GB Macs |

*(The "≈ 0.57 B/param" often quoted for "4-bit" is Q4_0's 4.5 bits; the Q4_K_M you'll actually download measures ~0.61 B/param from real file sizes.)*

## File sizes for common models

Measured from GGUF repos (ggml-org / bartowski / unsloth), 2026-11-02:

| Model (params) | FP16 | Q8_0 | Q6_K | Q5_K_M | Q4_K_M | Q3_K_M |
|---|---|---|---|---|---|---|
| 8B (Llama-3.1-8B) | 16.1 GB | 8.5 GB | 6.7 GB | 5.7 GB | 4.9 GB | 4.1 GB |
| 13B (Gemma-3-12B class) | 26 GB | 13.8 GB | 11.0 GB | 9.4 GB | 7.9 GB | 6.7 GB |
| 14B (Qwen2.5-14B) | 29.5 GB | 15.7 GB | 12.4 GB | 10.6 GB | 9.0 GB | 7.6 GB |
| 24B (Mistral-Small-24B) | 48.6 GB | 25.8 GB | 20.5 GB | 17.4 GB | 14.8 GB | 12.4 GB |
| 27B (Gemma-3-27B) | 53.2 GB | 28.3 GB | 22.5 GB | 19.1 GB | 16.2 GB | 13.5 GB |
| 32B (Qwen2.5-32B) | 65.6 GB | 34.8 GB | 27.7 GB | 23.6 GB | 20.0 GB | 16.8 GB |
| 70B (Llama-3.1-70B) | 141 GB | 75 GB | 59.5 GB | 50.4 GB | 42.7 GB | 34.9 GB |

MoE (size = total params, speed = active params):

| Model (total / active) | Q4_K_M | Q3_K_M / IQ3 | Q2 / IQ2 |
|---|---|---|---|
| Qwen3-30B-A3B (30.5B / 3.3B) | 18.6 GB | ~15 GB | ~11 GB |
| gpt-oss-120B (117B / 5.1B, native MXFP4) | ~63 GB (FP4) | — | — |
| Qwen3-235B-A22B (235B / 22B) | ~133 GB | ~110 GB | ~90 GB |
| DeepSeek-R1 / V3 (671B / 37B) | ~404 GB | ~330 GB | ~223 GB (IQ2_XXS) |

## Why K-quants beat naive int4

- **Super-blocks**: a K-quant groups 256 weights and stores per-group scales (and for Q4_K, per-group min) at high precision. Outlier tensors that destroy naive 4-bit get their scales preserved.
- **Mixed precision**: `Q4_K_M` uses Q6_K for the sensitive tensors (attention, some FFN) and Q4_K for the rest — that's the `_M` (medium). `_S` is all Q4_K, `_L` has more Q6_K.
- **imatrix (IQ / UD quants)**: an *importance matrix* computed on calibration data tells the quantizer which weights matter; bits are allocated accordingly. That's why `IQ4_XS` (4.25 bits) is often as good as `Q4_K_M` (4.85 bits). unsloth's `UD-*` dynamic quants do the same per-tensor.

## Quality guidance (dense models)

| Model size | Comfortable | Good default | Acceptable squeeze | Avoid |
|---|---|---|---|---|
| ≤ 8B | Q8_0 (if it fits) | Q4_K_M | Q4_0 / IQ4_XS | Q3 and below |
| 13–32B | Q5_K_M / Q6_K | **Q4_K_M** | Q3_K_M (short tasks) | Q2 |
| 70B | Q5_K_M (64+ GB) | **Q4_K_M** (48 GB) | Q3_K_M (32–36 GB) | Q2 except on Macs |

MoE is the exception: experts are redundant, so aggressive quants (IQ3/IQ2, mixed expert quants) cost little quality — that's how a 671B model runs in ~223 GB.

## Non-GGUF quantizations (vLLM / Transformers)

| Format | B/param | Notes |
|---|---|---|
| FP16 / BF16 | 2.0 | default for `vllm serve` |
| FP8 (E4M3) | 1.0 | near-fp16 quality; needs Ada (RTX 40) / Hopper / Blackwell (RTX 50) |
| GPTQ / AWQ | ~0.5–0.6 | calibration-based int4 for GPU serving; repo name says `GPTQ`/`AWQ` |
| bitsandbytes int8 | 1.0 | on-the-fly, no separate repo |

## My experiments

- (fill in: e.g. "A/B tested Q4_K_M vs Q6_K on Qwen2.5-14B for code tasks: Q6_K fixed occasional wrong imports, kept Q6_K")
