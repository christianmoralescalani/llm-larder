# MoE (Mixture of Experts)

**Summary.** MoE replaces each layer's dense feed-forward network with many small *expert* FFNs and a *router* that picks a few experts per token. Only the active experts are computed, so inference cost scales with **active** parameters, not total — but **all** experts must be loaded in memory. That's why a 671B-parameter model can decode at the speed of a ~37B dense model, and why "30B-A3B" means "30B stored, ~3B active".

## How it works

In a dense Transformer, every token goes through the same FFN in every layer. In an MoE layer:

```text
token → router (small learned classifier)
      → router scores all experts, picks top-k (e.g. 8 of 256)
      → token is processed by those k experts only
      → expert outputs are combined (weighted by router scores)
```

- **Router/gate**: a learned linear layer that outputs a score per expert; top-k are selected (often with a softmax over the chosen scores).
- **Experts**: independent FFNs, each much smaller than the dense FFN it replaces.
- **Shared expert** (DeepSeek style): one extra expert that *every* token uses, plus the routed ones. DeepSeek-V3: 256 routed + 1 shared, top-8 routed active.

## The numbers that matter

| Model | Total params | Active per token | Experts/layer | Top-k |
|---|---|---|---|---|
| DeepSeek-V3 / R1 | 671B | 37B | 256 + 1 shared | 8 |
| Qwen3-235B-A22B | 235B | 22B | 128 | 8 |
| Qwen3-30B-A3B | 30.5B | 3.3B | 128 | 8 |
| Qwen3-8B (dense, for contrast) | 8B | 8B | — | — |

(From the DeepSeek-V3 and Qwen3 technical reports.)

## What it means for local inference

1. **Memory = total params.** All experts are stored and must be loaded. A 671B Q4_K_M model is ~410 GB — far beyond one consumer GPU; it needs a big-RAM machine with CPU offload.
2. **Speed = active params.** Decode is memory-bandwidth-bound: `speed ≈ bandwidth ÷ active bytes per token`. 37B active at Q4 ≈ 23 GB read per token. On 50 GB/s of usable CPU+GPU mixed bandwidth that's ~2 tok/s; on a 1 TB/s GPU it'd be ~43 tok/s.
3. **The MoE sweet spot for home hardware**: small-total MoE models. Qwen3-30B-A3B at Q4_K_M (~19 GB) fits in a 24 GB card, and since only 3.3B params are active per token it decodes *faster* than a dense 8B on the same card. This is the single biggest reason "small MoE" models are the local-LLM value kings.
4. **Expert offloading**: with llama.cpp you can keep hot layers/experts on GPU and cold experts in RAM. Works, but every token that routes to a CPU expert costs a slow round-trip — expect a speed cliff, not a smooth gradient.
5. **Load balancing**: training must keep all experts busy (some models use auxiliary-loss-free balancing, DeepSeek-V3 §2.1.2). For inference you don't tune this, but it explains why expert counts look "too many".

## Dense vs MoE — quick decision

| Question | Dense wins | MoE wins |
|---|---|---|
| Fits entirely in VRAM? | Both fine | MoE: more capability per active byte |
| Doesn't fit in VRAM | Both slow (same math) | MoE slightly better (fewer active bytes) |
| Want max quality at fixed *speed* budget | — | MoE (more total knowledge per token) |
| Want simplest mental model of memory | params × bytes | total params × bytes, speed from active |

## My experiments

- (fill in: e.g. "ran Qwen3-30B-A3B Q4_K_M vs Qwen3-8B Q4_K_M side by side — 30B-A3B was ~2× faster and clearly smarter")
