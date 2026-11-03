# KV cache math: how context and batch eat VRAM

**Summary.** The KV cache stores the K and V vectors of every token already seen, so the model never recomputes attention. Its size is `2 × layers × kv_heads × head_dim × ctx × bytes` — it grows **linearly with context length and with batch size**, independently of the model weights. This is why 128K context on a 7–8B model can OOM a 16 GB card: the cache alone is ~17 GB. The levers are: shorter context, quantized cache (`q8_0` halves it, `q4_0` quarters it), GQA/MLA models (fewer KV heads), and controlling batch.

## The formula

```text
KV bytes = 2 × layers × kv_heads × head_dim × seq_len × bytes_per_elem
```

- `2` = K and V. `bytes_per_elem` = 2 (fp16/bf16), 1 (q8_0), 0.5 (q4_0) — llama.cpp `--cache-type-k/v`.
- Full derivation and per-model per-token costs: [00-foundations/cheatsheets/kv-cache-formula.md](../../00-foundations/cheatsheets/kv-cache-formula.md).

Per-token cache cost (fp16) for the models used in this repo:

| Model | KV heads | Per token | @ 4K | @ 32K | @ 128K |
|---|---|---|---|---|---|
| Llama-3.1-8B (32 layers) | 8 (GQA) | 128 KB | 0.52 GB | 4.3 GB | 17.2 GB |
| Qwen3-8B (36 layers) | 8 (GQA) | 144 KB | 0.58 GB | 4.6 GB | 18.4 GB |
| 13–14B class (48 layers) | 8 (GQA) | ~192 KB | 0.75 GB | 6.0 GB | 24 GB |
| Llama-3.1-70B (80 layers) | 8 (GQA) | 320 KB | 1.25 GB | 10.2 GB | 41 GB |
| DeepSeek-V3/R1 (61 layers) | MLA 512+64 | 69 KB | 0.28 GB | 2.2 GB | 9.0 GB |

GQA (grouped-query attention) is why modern 8B models only cost 128 KB/token instead of the 512 KB a same-size MHA model would. MLA (DeepSeek) compresses K/V into a latent vector and is what makes 128K context affordable on a 671B model.

## Worked example: why 128K on a 7–8B model OOMs

Llama-3.1-8B, Q4_K_M, on an 8 GB card (e.g. RTX 4060 / 5060):

```text
weights (Q4_K_M)        4.9 GB
KV cache @ 128K (fp16) 17.2 GB   ← 3.5× the weights
overhead                ~1.0 GB
─────────────────────────────────
total                  ~23.1 GB   → 3× the card. OOM.
```

Even on a 24 GB card it *barely* fits (23.1 GB) — and the prefill of 128K input tokens takes minutes and hammers compute. On a 16 GB card it simply doesn't fit.

**Fixes, in order of preference:**

1. **Use less context.** 32K costs 4.3 GB, not 17.2 GB. Most chats never need more than 8–16K.
2. **Quantize the cache.** `--cache-type-k q8_0 --cache-type-v q8_0` halves it (17.2 → 8.6 GB at 128K, ~negligible quality loss); `q4_0` quarters it (4.3 GB, small quality cost). Then 8B Q4 + 128K = 4.9 + 8.6 + 1 ≈ **14.5 GB → fits a 16 GB card**.
3. **Pick a GQA/MLA model** for the same budget (already standard in Llama 3 / Qwen / DeepSeek).

Runtime knobs: llama.cpp `--cache-type-k/v`, Ollama `OLLAMA_KV_CACHE=q8_0`, vLLM `--kv-cache-dtype fp8`.

## Batch size multiplies the cache

Each parallel sequence gets its own cache:

```text
KV total ≈ N_slots × per_token_cost × ctx
```

- llama.cpp `--parallel 4` on an 8B model at 8K: 4 × 1 GB = 4 GB of cache.
- vLLM `--max-num-seqs 16` at 4K on an 8B: 16 × 0.52 GB ≈ 8.3 GB.

Serving many users is a KV-cache game: vLLM's PagedAttention manages this cache in pages and does prefix caching (shared system prompts reuse cached K/V) so the effective cost per user is lower.

## Flash attention: what it does and doesn't do

- **Does**: computes attention without materializing the full N×N matrix → less *compute* and temporary memory during prefill, faster long-context prefill.
- **Does not**: shrink the KV cache. All K/V vectors are still stored for every token. `--flash-attn` helps prefill speed, not the VRAM budget.

## Sliding-window attention

Some models (Gemma 2, Mistral Small 3.1) only attend to the last W tokens. Their cache is bounded by W, not by total context — much cheaper, but the model genuinely cannot see beyond the window.

## Rules

1. **Compute the KV term before enabling long context** — on small models it can exceed the weights.
2. **Cache quantization is free lunch**: `q8_0` halves cache VRAM for ~no quality loss.
3. **Batch = multiplication**: N parallel slots ≈ N× the cache.
4. **GQA/MLA = more context per GB**: same model class, 4× cheaper cache than old MHA.
5. **Prefill of huge prompts is slow even when it fits** — 128K input takes minutes on consumer hardware.

## Sources

- [00-foundations/cheatsheets/kv-cache-formula.md](../../00-foundations/cheatsheets/kv-cache-formula.md) (formula, per-token table, MLA detail)
- Llama-3.1-8B / 70B, Qwen3-8B `config.json` via HF API, 2026-11-02.
- DeepSeek-V3 technical report §2.1.1 (MLA latent 512 + rope 64).

## My experiments

- (fill in: e.g. "8B Q4 + 128K ctx: fp16 cache OOM'd my 16 GB card at 23 GB; q8_0 cache brought it to 14.5 GB and it ran at ~60 t/s decode")
