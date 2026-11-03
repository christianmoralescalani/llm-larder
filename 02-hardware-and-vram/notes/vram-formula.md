# The VRAM formula: weights + KV cache + overhead

**Summary.** Everything a local LLM needs in GPU memory is the sum of three things: **weights** (fixed by model size × quantization), **KV cache** (grows with context length and batch size), and a small **overhead** (~1 GB). `VRAM needed ≈ weights + KV(ctx) + 1 GB`. Weights are the entry fee; the KV cache is the part people forget, and it is why "the file is 5 GB, why doesn't it fit in 8 GB?" happens. Worked examples below for 8B / 13B / 70B at Q4_K_M, Q8_0 and FP16.

## The formula

```text
VRAM needed ≈ weights + KV cache (ctx, batch) + overhead

weights     = params × bytes_per_weight          (see quantization-math.md)
KV cache    = 2 × layers × kv_heads × head_dim × ctx × bytes_per_elem   (see 00-foundations/cheatsheets/kv-cache-formula.md)
overhead    ≈ 0.5–1.5 GB (CUDA/Metal context, activations, workspace)
```

Two independent knobs:

- **Quantization** changes only the weights term.
- **Context length / batch** changes only the KV term — you can have a tiny quant and still OOM on long context.

## Weights by model size and quant

Measured GGUF file sizes (ggml-org / bartowski / unsloth repos, 2026-11-02):

| Model (params) | FP16 | Q8_0 | Q6_K | Q5_K_M | **Q4_K_M** | Q3_K_M |
|---|---|---|---|---|---|---|
| 8B (Llama-3.1-8B, 8.0B) | 16.1 GB | 8.5 GB | 6.7 GB | 5.7 GB | **4.9 GB** | 4.1 GB |
| 13B (e.g. Gemma-3-12B, Qwen2.5-14B) | 26 GB | 13.8 GB | 11.0 GB | 9.4 GB | **7.9 GB** | 6.7 GB |
| 70B (Llama-3.1-70B, 70.6B) | 141 GB | 75 GB | 59.5 GB | 50.4 GB | **42.7 GB** | 34.9 GB |

Per-param rule of thumb: **FP16 ≈ 2.0 GB/B · Q8_0 ≈ 1.06 GB/B · Q4_K_M ≈ 0.61 GB/B**.

## Worked examples

KV cache in FP16 (the default). Per-token cache cost: 8B ≈ 128 KB, 13B ≈ 160 KB, 70B ≈ 320 KB.

### 8B (Llama-3.1-8B)

| Quant | Weights | + 4K ctx | + 8K ctx | + 32K ctx | + 128K ctx |
|---|---|---|---|---|---|
| Q4_K_M | 4.9 GB | 6.4 GB | 6.9 GB | 10.2 GB | 23.1 GB |
| Q8_0 | 8.5 GB | 10.0 GB | 10.5 GB | 13.8 GB | 26.7 GB |
| FP16 | 16.1 GB | 17.6 GB | 18.1 GB | 21.4 GB | 34.3 GB |

→ **8B Q4_K_M fits an 8 GB card at 4–8K context** (6.4–6.9 GB total). At 32K it needs ~10 GB; at 128K it needs ~23 GB — the cache (17.2 GB) is *3.5× the weights*.

### 13B (representative 12–15B dense)

| Quant | Weights | + 4K ctx | + 8K ctx | + 32K ctx |
|---|---|---|---|---|
| Q4_K_M | 7.9 GB | 9.5 GB | 10.2 GB | 14.1 GB |
| Q8_0 | 13.8 GB | 15.4 GB | 16.1 GB | 20.0 GB |
| FP16 | 26 GB | 27.6 GB | 28.3 GB | 32.2 GB |

→ **13B Q4_K_M fits 16 GB at 8K comfortably, 32K just barely** (14.1 GB). 13B Q8_0 needs 16 GB at 4K and ~20 GB at 32K.

### 70B (Llama-3.1-70B)

| Quant | Weights | + 4K ctx | + 8K ctx | + 32K ctx |
|---|---|---|---|---|
| Q4_K_M | 42.7 GB | 44.9 GB | 46.2 GB | 53.9 GB |
| Q8_0 | 75 GB | 77.2 GB | 78.5 GB | 86.2 GB |
| FP16 | 141 GB | 143.2 GB | 144.5 GB | 152.2 GB |

→ **70B Q4_K_M needs ~48 GB at 8K** (2×24 GB cards, or a 48/64 GB Mac). 70B Q8_0 needs ~96 GB. 70B FP16 needs ~144 GB (2×80 GB datacenter GPUs).

## Rules of thumb

1. **Budget 25–40% of VRAM for KV + overhead**, not just weights. A 24 GB card runs ~16–18 GB of weights comfortably, ~20 GB with a small context.
2. **FP16 = 2 GB per 1B params, Q8_0 ≈ 1.06, Q4_K_M ≈ 0.61.** Multiply by your param count, add KV, add 1 GB.
3. **Context is not free**: every 8K tokens on an 8B model costs ~1 GB of FP16 cache.
4. **Batching multiplies the cache**: N parallel slots ≈ N × per-sequence KV (see [kv-cache-math.md](kv-cache-math.md)).
5. **MoE models**: size is set by *total* params (experts all live in memory), speed by *active* params. Qwen3-30B-A3B is 18.6 GB at Q4_K_M (30.5B total) but decodes like a ~3.3B model.
6. **Verify, don't trust**: the runtime prints what it actually uses (llama.cpp `offloaded X/Y layers`, `ollama ps`, `nvidia-smi`).

## Common mistakes

- Sizing only on the file size and forgetting KV → OOM at long context.
- Forgetting the ~1 GB overhead → "it should fit" doesn't.
- Assuming FP16 is needed for quality → Q8_0 is near-lossless at half the memory.
- Ignoring batch: 4 parallel chats on an 8B model at 8K needs ~5 GB of cache, not 1 GB.

## Sources

- KV cache formula and per-model per-token costs: [00-foundations/cheatsheets/kv-cache-formula.md](../../00-foundations/cheatsheets/kv-cache-formula.md)
- Quant bytes/weight and file sizes: [quantization-math.md](quantization-math.md)
- Llama-3.1-8B / 70B `config.json` (layers, KV heads, head_dim) via HF API, 2026-11-02.
- GGUF file sizes: ggml-org / bartowski / unsloth repos on HF, 2026-11-02.

## My experiments

- (fill in: e.g. "ran Llama-3.1-8B Q4_K_M at 8K on my 12 GB card: `nvidia-smi` showed 7.1 GB used — matches the 6.9 GB estimate")
