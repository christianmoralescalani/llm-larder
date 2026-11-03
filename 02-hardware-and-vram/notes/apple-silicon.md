# Apple Silicon: unified memory for local LLMs

**Summary.** Macs run big LLMs because of **unified memory**: CPU and GPU share one physical RAM pool, and the GPU can address most of it — no PCIe copy, no separate VRAM. The catch is **bandwidth**: even a top M5 Ultra (1200 GB/s) is below an RTX 4090 (1008 GB/s) only at the very top, and a base M4 (120 GB/s) is 8× slower than a 4090. Decode speed follows `tok/s ≈ bandwidth × 0.6–0.8 ÷ model_bytes`. The buying rule: **memory capacity first, bandwidth tier second, generation third** — and watch the bandwidth *bins* (the same chip name can ship with 25% less bandwidth in cheaper configs).

## Why Macs run big models

- **One memory pool.** A 128 GB Mac gives the GPU ~96–100 GB of usable memory (macOS keeps some for the system). A 128 GB *discrete* GPU setup would cost several datacenter cards.
- **No copy overhead.** Weights live in the same RAM the CPU reads; offloading is a non-issue — there is only one place to put them.
- **The tradeoff: bandwidth.** LPDDR5X unified memory is slower than GDDR6X/7. That's why a 70B Q4 decodes at ~10 t/s on an M4 Max but ~20 t/s on 2× RTX 4090.
- **Practical GPU budget ≈ 75% of physical RAM.** Go beyond that and macOS starts swapping to SSD (below).

## Bandwidth table (verified 2026-11-02)

| Chip | RAM options | Bandwidth | Notes |
|---|---|---|---|
| M1 (2020) | 8 / 16 / 24 GB | 68 GB/s | |
| M1 Pro | 16 / 32 GB | 200 GB/s | |
| M1 Max | 32 / 64 GB | 400 GB/s | |
| M1 Ultra | 64 / 128 GB | 800 GB/s | |
| M2 (2022) | 8 / 16 / 24 GB | 101 GB/s | |
| M2 Pro | 16 / 32 GB | 200 GB/s | |
| M2 Max | 32 / 64 / 96 GB | 400 GB/s | |
| M2 Ultra | 64 / 128 / 192 GB | 800 GB/s | first Mac to run 70B Q8 comfortably |
| M3 (2023) | 8 / 16 / 24 GB | 101 GB/s | |
| M3 Pro | 18 / 36 GB | 150 GB/s | **bandwidth regression vs M2 Pro (200)** |
| M3 Max | 36 / 48 / 64 / 128 GB | 300 (36 GB) / 400 (48+) | **36 GB config is binned down to 300 GB/s** |
| M3 Ultra | 96 / 512 GB | 819 GB/s | |
| M4 (2024) | 16 / 24 / 32 GB | 120 GB/s | |
| M4 Pro | 24 / 48 GB | 273 GB/s | |
| M4 Max | 36 / 48 / 64 / 128 GB | 410 (36 GB) / 546 (48+) | **36 GB config binned down to 410** |
| M5 (2025) | 16 / 24 / 32 GB | 153 GB/s | |
| M5 Pro | 24 / 48 / 64 GB | 307 GB/s | |
| M5 Max | 36 / 48 / 64 / 128 GB | 460 (36 GB) / 614 (48+) | Neural Accelerators → much faster prefill |
| M5 Ultra | 96 / 256 / 512 GB | 1200 GB/s | top consumer Mac; 400B+ MoE territory |

The binned configs are the trap: an M3 Max or M4 Max with the *cheapest* RAM option loses 25% bandwidth for no visible benefit in the spec sheet. When buying a Max chip, pay for the full-bandwidth RAM tier.

## Estimating tokens/sec

```text
decode tok/s ≈ bandwidth (GB/s) × 0.6–0.8 ÷ model_bytes (Q4_K_M)
```

| Hardware (bandwidth) | 8B Q4 (4.9 GB) | 14B Q4 (9.0 GB) | 32B Q4 (20 GB) | 70B Q4 (42.7 GB) |
|---|---|---|---|---|
| M1 8/16/24 (68) | ~10–15 | ~5–8 | — | — |
| M2 (101) | ~15–20 | ~8–12 | — | — |
| M4 (120) | ~18–22 | ~10–14 | — | — |
| M2/M3 Max 400 | ~40–50 | ~25–30 | ~12–15 | ~7–9 |
| M4 Max 546 | ~55–70 | ~35–45 | ~18–25 | ~10–12 |
| M5 Max 614 | ~60–80 | ~40–50 | ~20–28 | ~12–15 |
| M1/M2 Ultra 800 | ~80–100 | ~50–60 | ~30–35 | ~13–18 |
| M3 Ultra 819 | ~80–100 | ~50–60 | ~30–35 | ~15–18 |
| M5 Ultra 1200 | ~120–150 | ~80–100 | ~50–60 | ~20–25 |

*(Rough: roofline × 0.6–0.8 efficiency, cross-checked against community benchmarks — llama.cpp discussion #4167, kunalganglani.com/llm-benchmarks. MoE models use active-param bytes: Qwen3-30B-A3B on an M4 Max ≈ 60–100 t/s.)*

## What fits (by RAM tier)

Q4_K_M weights + 8K KV + 1 GB overhead, staying under ~75% of RAM:

| RAM | Comfortable | Stretch |
|---|---|---|
| 16 GB | 8B Q4 + 8K, 3B Q8 | 14B Q4 at 4K (tight) |
| 24–32 GB | 14B Q4 + 32K, 8B Q8 + 32K | 24B Q3, 27B Q4 at 4K |
| 48–64 GB | 32B Q4 + 32K, 24B Q8, 30B-A3B MoE Q4 + 32K | 70B Q2 at 4K |
| 96–128 GB | **70B Q4 + 32K (54 GB)**, 70B Q5, 235B MoE Q2 | 70B Q6 (62 GB) |
| 192 GB (M2 Ultra) | 70B Q8 (86 GB), 235B MoE Q3 (~110 GB) | 405B Q2 (156 GB, tight) |
| 512 GB (M3/M5 Ultra) | 405B Q2–Q3, 235B Q4 + big ctx, DeepSeek-R1 IQ2 (223 GB) | 671B IQ2 |

## Swapping: the silent killer

When total usage exceeds physical RAM, macOS pages memory out to the SSD:

- With `mmap` (default), weights get evicted and re-faulted from disk **per token** → tok/s collapses from ~10 to <1.
- Symptom: speed is fine for a few tokens, then crawls, then the fan kicks in (SSD).
- **Rule: model file + KV cache + OS ≤ ~75% of physical RAM.** On a 128 GB Mac that's ~96 GB of model+cache — a 70B Q4 (42.7 GB) + 32K KV (10.2 GB) is safe; a 70B Q8 + 32K (86 GB) is at the edge.
- Check: Activity Monitor → Memory → "Memory Pressure" (yellow/red = swapping).

## MLX vs llama.cpp (Metal)

| | MLX (`mlx-lm`) | llama.cpp (Metal) |
|---|---|---|
| Origin | Apple's ML framework | ggml/llama.cpp community |
| Strengths | best prefill on M4/M5 (neural accelerators), NVFP4 on M5, clean Python API | most quants (all GGUF), most mature, Ollama/LM Studio use it under the hood |
| When | M5-generation Macs, Python workflows, prompt-heavy agentic use | everything else, GGUF ecosystem, max quant choice |

Both are fine; the difference is a few percent on decode, larger on prefill (M5: MLX up to ~4× faster prompt processing per Apple).

## Buying advice

- **Capacity first**: 32 GB is the minimum serious tier; 64 GB runs 32B comfortably; 128 GB runs 70B Q4. A 16 GB M5 is a faster 16 GB machine — it still can't run 70B.
- **Watch the bandwidth bin**: on M3 Max / M4 Max / M5 Max, the cheapest RAM config is binned down (300/410/460 GB/s). Pay for the 48 GB+ tier.
- **M3 Pro regression**: 150 GB/s vs M2 Pro's 200 — for LLMs the M2 Pro is the better 32 GB chip.
- **Best 70B laptop**: 128 GB M4 Max / M5 Max (~10–15 t/s).
- **Best Mac overall for local LLMs**: Mac Studio M5 Ultra 512 GB (1200 GB/s) — runs 405B-class and DeepSeek-R1 IQ2 at usable speed; M3 Ultra 512 GB (819 GB/s) is the used-market alternative.

## Sources

- Apple tech specs: [MacBook Pro (M4 Pro/M4 Max)](https://support.apple.com/en-us/121553), [Mac Studio (M5 Max/M5 Ultra)](https://support.apple.com/en-us/128107), Apple newsroom M5 Pro/Max announcement (2026-03) — checked 2026-11-02
- llama.cpp discussion #4167 (Apple Silicon benchmark collection)
- kunalganglani.com/llm-benchmarks (measured tok/s database)
- [vram-formula.md](vram-formula.md) for the fit math, [vram-vs-ram-cpu.md](vram-vs-ram-cpu.md) for the bandwidth→tok/s model

## My experiments

- (fill in: e.g. "M3 Max 128 GB: Llama-3.1-70B Q4_K_M = 9 t/s decode, 350 t/s prefill; memory pressure stayed green at 32K ctx")
