# GPU guide: consumer cards for local LLMs

**Summary.** For local LLMs the GPU spec sheet ranks in a fixed order: **1) VRAM capacity** (binary — the model+cache fit or it doesn't), **2) memory bandwidth** (sets decode tok/s), **3) compute** (sets prefill speed and batching), **4) FP8 support** (Ada/Hopper/Blackwell: near-fp16 quality at 1 B/param). A 24 GB card with 936 GB/s (used RTX 3090) beats a 16 GB card with 960 GB/s for model range, and two 24 GB cards beat one 32 GB card for 70B. Tables below for the RTX 30/40/50 series and what each VRAM tier runs comfortably.

## What matters, in order

| Spec | What it decides | Example |
|---|---|---|
| **VRAM capacity** | which models fit (weights + KV + 1 GB) | 24 GB → 32B Q4 + 32K ctx |
| **Memory bandwidth** | decode tok/s (`bandwidth ÷ model_bytes`) | 936 vs 448 GB/s ≈ 2× tok/s on the same model |
| **Compute (TFLOPs)** | prefill speed (TTFT), batch throughput | 5090 prefills 128K prompts far faster than a 3060 |
| **FP8 / FP4** | near-fp16 quality at 1 (or 0.5) B/param | RTX 40/50 series; not on RTX 30 |

Decode is bandwidth-bound, prefill is compute-bound — see [00-foundations/notes/inference-pipeline.md](../../00-foundations/notes/inference-pipeline.md).

## RTX 30 series (Ampere, 2020–21)

| Card | VRAM | Bus | Bandwidth | TDP | Notes |
|---|---|---|---|---|---|
| RTX 3050 | 8 GB | 128-bit | 224 GB/s | 130 W | 8B Q4 only, slow |
| RTX 3060 | 8 / 12 GB | 192-bit | 360 GB/s | 170 W | 12 GB version = best budget 12 GB |
| RTX 3060 Ti | 8 / 16 GB | 256-bit | 320 GB/s | 200 W | 16 GB version rare/used |
| RTX 3070 | 8 GB | 256-bit | 448 GB/s | 220 W | 8B Q8 / 13B Q4 |
| RTX 3070 Ti | 8 GB | 256-bit | 608 GB/s | 290 W | |
| RTX 3080 | 10 / 12 GB | 320-bit | 760 GB/s | 320 W | |
| RTX 3080 Ti | 12 GB | 320-bit | 912 GB/s | 350 W | |
| **RTX 3090** | **24 GB** | 384-bit | **936 GB/s** | 350 W | **the value king; NVLink-capable** |
| RTX 3090 Ti | 24 GB | 384-bit | 1008 GB/s | 450 W | |

## RTX 40 series (Ada, 2022–24)

| Card | VRAM | Bus | Bandwidth | TDP | Notes |
|---|---|---|---|---|---|
| RTX 4060 | 8 GB | 128-bit | 272 GB/s | 115 W | 8B Q4 |
| RTX 4060 Ti | 8 / 16 GB | 128-bit | 288 GB/s | 160 W | **16 GB version = budget sweet spot** |
| RTX 4070 | 12 GB | 192-bit | 504 GB/s | 200 W | |
| RTX 4070 Super | 12 GB | 192-bit | 504 GB/s | 220 W | |
| RTX 4070 Ti | 12 GB | 192-bit | 504 GB/s | 260 W | |
| RTX 4070 Ti Super | 16 GB | 256-bit | 672 GB/s | 285 W | 16 GB at good bandwidth |
| RTX 4080 | 16 GB | 256-bit | 717 GB/s | 320 W | |
| **RTX 4090** | **24 GB** | 384-bit | **1008 GB/s** | 450 W | fastest 24 GB consumer card |

## RTX 50 series (Blackwell, 2025–26)

| Card | VRAM | Bus | Bandwidth | TDP | Notes |
|---|---|---|---|---|---|
| RTX 5050 | 8 GB | 128-bit | 320 GB/s | 150 W | |
| RTX 5060 | 8 GB | 128-bit | 448 GB/s | 145 W | |
| RTX 5060 Ti | 8 / 16 GB | 128-bit | 448 GB/s | 180 W | **16 GB version = new budget sweet spot** |
| RTX 5070 | 12 GB | 192-bit | 672 GB/s | 250 W | |
| RTX 5070 Ti | 16 GB | 256-bit | 896 GB/s | 300 W | same 16 GB as 5080, 896 vs 960 GB/s |
| RTX 5080 | 16 GB | 256-bit | 960 GB/s | 360 W | |
| **RTX 5090** | **32 GB** | 512-bit | **1792 GB/s** | 575 W | fastest consumer card; 70B Q3–Q4 territory |

All RTX 40/50 cards support **FP8** (native tensor cores) — a 1 B/param format with near-fp16 quality; RTX 50 adds FP4. RTX 30 does **not** support FP8.

## What each VRAM tier runs comfortably

Dense models at Q4_K_M, 8K context, single user (KV + overhead included — full math in [vram-formula.md](vram-formula.md)):

| VRAM | Comfortable | Stretch / tight |
|---|---|---|
| 8 GB | 7–8B Q4_K_M (4–8K ctx) | 3B Q8_0 at 4K (borderline) |
| 12 GB | 13–14B Q4_K_M + 8K, 8B Q8_0 + 8K | 8B Q8_0 + 32K (q8_0 cache) |
| 16 GB | 13B Q8_0 + 4K, 14B Q4_K_M + 8K, 8B Q4 + 32K | 24B Q4_K_M at 4K (16.5 GB) |
| 24 GB | **24B Q4_K_M + 32K, 32B Q4_K_M + 8K, 30B-A3B MoE Q4 + 32K** | 27B Q4_K_M + 32K (~26 GB) |
| 32 GB | 32B Q6_K + 8K, 70B Q2_K + 4K (tight) | 30B-A3B Q8_0 + 8K (34.2 GB) |
| 48 GB (2×24) | **70B Q4_K_M + 8K, 32B Q8_0 + 32K** | 70B Q5_K_M + 8K (53.5 GB) |
| 64 GB (2×32 / 4×16) | 70B Q5–Q6_K, 106B MoE Q4 (tight) | 70B Q8 needs 96 |
| 96 GB (3×32 / 4×24) | 70B Q8_0 + 8K, 235B MoE Q2_K (tight) | 235B Q3_K_M needs 128 |

## Buying advice (2026)

- **Best value overall: used RTX 3090 24 GB** (~$700–900). 936 GB/s + 24 GB + NVLink; still runs 32B Q4 + 32K at ~40–60 t/s.
- **New budget: 16 GB cards.** RTX 5060 Ti 16 GB / 4060 Ti 16 GB (cheapest), 5070 Ti 16 GB (896 GB/s, best 16 GB speed under the 5080). The 16 GB ceiling is the same on all of them — bandwidth decides speed.
- **5070 Ti vs 5080**: identical 16 GB, 896 vs 960 GB/s (~7% speed). Buy the cheaper one.
- **RTX 5090 32 GB**: the only 32 GB consumer card; 1792 GB/s makes 32B models ~2× faster than a 3090. 575 W — plan the PSU.
- **Two 24 GB cards** (2×3090 / 2×4090) beat one 5090 for 70B Q4 (48 GB > 32 GB). See [multi-gpu.md](multi-gpu.md).
- **AMD**: RX 7900 XTX 24 GB (ROCm, solid on Linux, Windows support improving), RX 7800 XT 16 GB. Slightly less bandwidth than NVIDIA equivalents, no FP8.
- **Intel**: Arc B580 12 GB (SYCL backend in llama.cpp/Ollama) — fine for 8B, experimental for more.
- **Laptops**: 16 GB VRAM minimum (RTX 4070/5070 laptop), 24 GB (RTX 4090/5090 laptop) for 32B-class. Laptop cards are power-limited; expect ~50–70% of desktop tok/s.

## Sources

- NVIDIA product pages: [RTX 50 series](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/), [RTX 40 series](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/), [RTX 30 series](https://www.nvidia.com/en-us/geforce/graphics-cards/30-series/) (specs checked 2026-11-02)
- TechPowerUp GPU database (bandwidth/TDP cross-check), 2026-11-02
- [vram-budget.md](../cheatsheets/vram-budget.md) for the fit table, [multi-gpu.md](multi-gpu.md) for 2-card setups

## My experiments

- (fill in: e.g. "llama-bench on my 4060 Ti 16 GB: Llama-3.1-8B Q4_K_M pp512 = 850 t/s, tg128 = 38 t/s")
