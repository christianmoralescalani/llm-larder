# Cheatsheet — KV cache formula

**TL;DR.** `KV cache = 2 × layers × kv_heads × head_dim × seq_len × bytes`. The `2` = K and V. GQA/MLA models have few kv_heads, so their cache is much smaller than MHA models at the same context. Compute this *before* enabling long context — on small models the cache can exceed the weights.

## The formula

```text
KV bytes = 2 × layers × kv_heads × head_dim × seq_len × bytes_per_elem
```

- `bytes_per_elem` = 2 (fp16/bf16), 1 (q8_0), 0.5 (q4_0) — llama.cpp `--cache-type-k/v`.
- `seq_len` = context size (prompt + generated tokens).
- Note: `kv_heads × head_dim` = the KV dimension per layer (often much smaller than `hidden`).

## Worked examples (fp16 cache)

Per-token cache cost first: `2 × layers × kv_heads × head_dim × 2 bytes` (the leading 2 = K+V).

| Model | Per-token cache | @ 4K ctx | @ 32K ctx | @ 128K ctx |
|---|---|---|---|---|
| Llama-3.1-8B (GQA: 32 layers, 8 KV heads, 128) | 128 KB | 0.52 GB | 4.3 GB | 17.2 GB |
| Qwen3-8B (GQA: 36 layers, 8 KV heads, 128) | 144 KB | 0.58 GB | 4.6 GB | 18.4 GB |
| Hypothetical 8B MHA (32 layers, 32 KV heads, 128) | 512 KB | 2.0 GB | 16.0 GB | 64.0 GB |
| DeepSeek-V3 (MLA: 61 layers, 512 latent + 64 rope) | 69 KB | 0.28 GB | 2.2 GB | 9.0 GB |

MLA detail: instead of full K/V vectors, each layer caches a compressed 512-dim latent + 64-dim rope key per token: `61 × (512+64) × 2 bytes = 70,272 B/token`. **MLA is what makes 128K context on a 671B model affordable** — that's the whole trick.

## Quick per-token cost table (fp16)

`bytes per token in cache = 2 × layers × kv_heads × head_dim × 2`

| Model | B/token | @ 32K | @ 128K |
|---|---|---|---|
| Llama-3.1-8B | 131,072 (128 KB) | 4.3 GB | 17.2 GB |
| Qwen3-8B | 147,456 (144 KB) | 4.6 GB | 18.4 GB |
| Qwen3-30B-A3B (48 layers, 4 KV, 128) | 98,304 (96 KB) | 3.1 GB | 12.3 GB |

## Rules

1. **VRAM budget** = weights + KV cache + ~1 GB overhead. If `weights + KV(ctx) > VRAM`, lower ctx, quantize the cache, or use a GQA/MLA model.
2. **Cache quantization** (`--cache-type-k q8_0 --cache-type-v q8_0`): halves cache VRAM, ~negligible quality loss. `q4_0`: quarters it, small quality cost.
3. **Batching multiplies the cache**: N parallel slots ≈ N × per-sequence cache (llama.cpp `--parallel`, vLLM `--max-num-seqs`).
4. **Long context ≠ free**: 128K on an 8B model = ~17–18 GB of cache alone, plus a slow prefill.
5. **Sliding window attention** (some models, e.g. Gemma 2, Mistral Small 3.1): cache is bounded by the window — much cheaper, but the model can't attend beyond the window.

## Verified on

- Llama-3.1-8B config (32 layers, 8 KV heads, head_dim 128, ctx 131072) via `unsloth/Llama-3.1-8B` config.json (HF API), 2026-10-27.
- Qwen3-8B config (36 layers, 8 KV heads, head_dim 128) and Qwen3-30B-A3B (48 layers, 4 KV heads, head_dim 128) via HF API, 2026-10-27.
- DeepSeek-V3 MLA latent sizes (512 + 64) from DeepSeek-V3 technical report §2.1.1, 2026-10-27.
