# Cheatsheet — "I have X GB → which models/quants fit"

**TL;DR.** `VRAM ≈ params × bytes/weight + KV(ctx) + 1 GB`. Bytes/weight: **FP16 = 2.0 · Q8_0 = 1.06 · Q6_K = 0.84 · Q4_K_M = 0.61 · Q3_K_M = 0.49**. Keep weights ≤ ~75% of your VRAM so context fits. 8B ≈ 128 KB of KV per token (fp16), 70B ≈ 320 KB — long context can cost more than the weights.

## Quick math

```text
weights (GB)  = params (B) × bytes/weight          # 8B @ Q4_K_M = 8 × 0.61 ≈ 4.9 GB
KV (GB)       = per_token_KB × ctx / 1M            # 8B @ 32K = 128 × 32768 / 1M ≈ 4.3 GB
total         = weights + KV + 1
```

Per-token KV (fp16): 3B ≈ 64 KB · 8B ≈ 128 KB · 13–14B ≈ 192 KB · 32B ≈ 256 KB · 70B ≈ 320 KB.
Cache quantization: `q8_0` halves it, `q4_0` quarters it. Batch N ≈ N× the KV.

## Decision table

Dense models, Q4_K_M unless noted, 8K context, single user (totals include KV + 1 GB overhead):

| Your VRAM | Comfortable | Stretch / tight |
|---|---|---|
| **8 GB** | 7–8B Q4_K_M + 4–8K (6.4–6.9 GB) | 3B Q8_0 + 4K (borderline) |
| **12 GB** | 13–14B Q4_K_M + 8K (10.2–11.6 GB), 8B Q8_0 + 8K (10.5 GB) | 8B Q8_0 + 32K with q8_0 cache (11.7 GB) |
| **16 GB** | 13B Q8_0 + 4K (15.4 GB), 14B Q4_K_M + 8K (11.6 GB), 8B Q4_K_M + 32K (10.2 GB) | 24B Q4_K_M at 4K (16.5 GB), 14B Q4 + 32K (16.3 GB) |
| **24 GB** | **24B Q4_K_M + 32K (21.8 GB)**, 32B Q4_K_M + 8K (23.1 GB), 30B-A3B MoE Q4 + 32K (22.7 GB), 14B Q8_0 + 32K (23 GB) | 27B Q4_K_M + 32K (~26 GB) |
| **32 GB** | 32B Q6_K + 8K (30.8 GB), 70B Q2_K + 4K (29.4 GB, tight), 24B Q8_0 + 32K (32.8 GB, tight) | 30B-A3B Q8_0 + 8K (34.2 GB); 70B Q4 needs 48 |
| **48 GB** (2×24) | **70B Q4_K_M + 8K (46.2 GB)**, 32B Q8_0 + 32K (44.2 GB), 30B-A3B Q8_0 + 32K (36.5 GB) | 70B Q5_K_M + 8K (53.5 GB) |
| **64 GB** (2×32 / 4×16) | 70B Q6_K + 8K (62.6 GB), 70B Q5_K_M + 32K (61.6 GB) | 106B MoE Q4_K_M (~65 GB); 70B Q8 needs 96 |
| **96 GB** (3×32 / 4×24) | **70B Q8_0 + 8K (78.5 GB)**, 235B MoE Q2_K (~90 GB, tight) | 235B Q3_K_M (~110 GB, needs 128) |
| **128 GB** (Mac) | 70B Q8_0 + 4K (77.2 GB), 70B Q6_K + 32K (73.8 GB), 235B MoE Q3_K_M + 8K (~112 GB) | 405B IQ2_XXS (~105 GB) |
| **192 GB** (M2 Ultra) | 70B Q8_0 + 32K (86.2 GB), 235B MoE Q3_K_M + 32K (~115 GB), 405B Q2_K + 8K (161 GB) | 405B Q3_K_M (~203 GB) |
| **512 GB** (M3/M5 Ultra) | 405B Q2_K–Q3_K_M + 32K (173–215 GB), 235B Q4_K_M + 128K (159 GB), DeepSeek-R1 IQ2_XXS + 32K (226 GB) | 671B Q3/IQ3 (~300 GB) |

MoE shortcut (size by total params, speed by active): Qwen3-30B-A3B Q4 = 18.6 GB (runs on 24 GB with 32K ctx, decodes like a 3B). gpt-oss-120B ≈ 63 GB (fits 96 GB). Qwen3-235B-A22B Q4 ≈ 133 GB.

## KV cache reminders

| Model | Per token (fp16) | 4K | 32K | 128K |
|---|---|---|---|---|
| 8B (Llama-3.1) | 128 KB | 0.5 GB | 4.3 GB | 17.2 GB |
| 13–14B | ~192 KB | 0.75 GB | 6.0 GB | 24 GB |
| 70B (Llama-3.1) | 320 KB | 1.25 GB | 10.2 GB | 41 GB |
| DeepSeek-V3/R1 (MLA) | 69 KB | 0.28 GB | 2.2 GB | 9.0 GB |

- 128K on an 8B model = 17.2 GB of cache — more than the Q4 weights (4.9 GB). Use `q8_0` cache (8.6 GB) or shorter ctx.
- Batch: N parallel slots ≈ N× the KV above.

## Verify on your machine

```bash
nvidia-smi                          # VRAM free/used, card model
ollama ps                           # what's loaded, VRAM used
# llama.cpp log line to watch:
#   "offloaded 80/80 layers to GPU"   ← fewer = CPU fallback, slower
llama-bench -m model.gguf -n 128    # measured pp/tg tok/s
```

## Rules

1. Weights ≤ ~75% of VRAM, else context won't fit.
2. Compute the KV term before enabling long context.
3. `q8_0` cache = free 2× context.
4. Doesn't fit? → MoE model, or partial offload (see `notes/vram-vs-ram-cpu.md`), or another quant (see `notes/quantization-math.md`).

## Verified on

- GGUF file sizes: ggml-org / bartowski / unsloth repos (HF), 2026-11-02.
- KV per-token costs: Llama-3.1-8B/70B, Qwen3-8B `config.json` via HF API, 2026-11-02.
- GPU/Mac specs: NVIDIA product pages, Apple tech specs, 2026-11-02.
