# Inference pipeline

**Summary.** Running an LLM locally is two very different phases: **prefill** (process the whole prompt at once — compute-bound, fast per token) and **decode** (generate one token at a time — memory-bandwidth-bound, slow per token). Almost every performance decision in local LLMs (quantization, batching, offloading, speculative decoding) is a consequence of this split. Understanding prefill/decode is the single most useful mental model for interpreting benchmarks.

## The two phases

```text
┌─────────────────────────────┐     ┌──────────────────────────────┐
│ PREFILL (prompt processing) │     │ DECODE (token generation)    │
├─────────────────────────────┤     ├──────────────────────────────┤
│ input:  N tokens at once    │     │ input:  1 token at a time    │
│ work:   big matrix mults    │     │ work:   small matrix mults   │
│ bound by: COMPUTE (FLOPs)   │     │ bound by: MEMORY BANDWIDTH   │
│ speed:  1000s of tok/s      │     │ speed:  5–100 tok/s          │
│ output: KV cache filled     │     │ output: 1 new token each step│
└─────────────────────────────┘     └──────────────────────────────┘
```

Each decode step: read the (active) weights from memory, multiply by one token's vector, append the new token's K/V to the cache, sample, repeat. The weights are read **every step** and each weight is used for ~1 token — so the bottleneck is how fast you can *read* weights, not how fast you can multiply.

### The bandwidth formula (decode)

```text
decode speed (tok/s) ≈ memory bandwidth ÷ bytes read per token
```

- **Bytes per token ≈ active params × bytes/weight** (for a dense model, all params are active).
- Example: 8B model at Q4_K_M ≈ 8.03B × 0.61 B ≈ 4.9 GB read per token. On a 4090 (~1 TB/s): ~200 tok/s ceiling; a real llama.cpp run lands lower (overheads, non-weight traffic).
- Example: 37B-active MoE at Q4 ≈ 23 GB/token → ~43 tok/s on the same card.

This is why **quantization barely changes decode speed** (fewer bytes → faster, proportionally) and why **MoE active count dominates speed**.

### Prefill speed

Prefill is compute-bound: all N prompt tokens are processed in parallel (causal masking keeps it one-directional). Speed scales with GPU FLOPs, so a 32K prompt on a 4090 might take ~1–3 s while the same prompt's 100-token answer takes ~5 s. Long prompts = long "time to first token" (TTFT).

## Where the time goes (typical local run)

| Phase | What happens | Dominant cost |
|---|---|---|
| Load | read weights from disk → VRAM/RAM | disk bandwidth, one-time |
| Prefill | process prompt, build KV cache | GPU compute |
| Decode ×N | one token per step | memory bandwidth (weights + cache reads) |
| (Optional) Speculation | draft k tokens, verify in one prefill-like pass | can multiply decode throughput |

## Levers you actually pull

| Lever | Affects | Why |
|---|---|---|
| Quantization (Q8→Q4) | decode speed ↑, quality ↓ slightly | fewer bytes read per token |
| Bigger batch / continuous batching | prefill throughput ↑ | amortizes weight reads across requests |
| GPU offload of all layers | decode speed ↑↑ | GPU bandwidth ≫ CPU bandwidth |
| KV cache quantization (q8_0/q4_0) | VRAM ↓ | cache is the second-biggest memory user |
| Speculative decoding / MTP | decode speed ↑ | verifies several tokens per pass |
| Flash attention | prefill speed ↑, VRAM ↓ | fuses attention ops, avoids materializing N×N matrix |
| Smaller active params (MoE) | decode speed ↑↑ | fewer bytes per token |

## Reading benchmarks (llama-bench, etc.)

- `pp N` (prompt processing): prefill speed, tok/s — compute-bound.
- `tg N` (token generation): decode speed, tok/s — bandwidth-bound.
- A good local setup: `tg` in the tens of tok/s for 7–8B class; `pp` in the hundreds–thousands.
- If `tg` is 2–5× lower than the bandwidth formula predicts: not all layers on GPU, slow CPU offload path, or CPU RAM as the bottleneck.

## My experiments

- (fill in: e.g. "llama-bench on my 4090: Qwen3-8B Q4_K_M → pp 2400 tok/s, tg 48 tok/s; formula predicted ~55 — gap explained by …")
