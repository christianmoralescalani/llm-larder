# What are parameters

**Summary.** A parameter (weight) is one learned number inside the model. "7B" means the model holds about 7 billion such numbers. Parameters are the model's *memory* — all its knowledge, style and behavior are encoded in them. More parameters generally means more capability (scaling laws), but capability ≠ parameter count: data quality, training recipe and post-training matter a lot. And the dense-vs-sparse (MoE) distinction changes what the number means for *speed* — a 671B MoE model can run at the speed of a ~37B dense model, if you can load all 671B.

## What a weight actually is

Every layer of a Transformer is a set of matrices (attention projections, feed-forward networks, embeddings). A forward pass is a chain of matrix multiplications:

```text
token ids → embeddings → [attention + FFN] × N layers → final vector → logits over the vocabulary
```

Each matrix cell is one parameter. The total count is the sum over all matrices.

## Why "7B" means 7 billion — and what that costs

File size ≈ parameters × bytes per weight:

| Model | Params | FP16 (2 B/p) | Q4_K_M (~0.61 B/p) |
|---|---|---|---|
| Qwen2.5-7B | 7.6B | ~15.2 GB | ~4.7 GB |
| Llama-3.1-8B | 8.0B | ~16.0 GB | ~4.9 GB |
| Qwen3-30B-A3B (MoE) | 30.5B | ~61 GB | ~18.7 GB |
| DeepSeek-V3 (MoE) | 671B | ~1.3 TB | ~410 GB |

(Real file sizes verified from HF, see [../cheatsheets/quantization-levels.md](../cheatsheets/quantization-levels.md).)

**Rough sanity formula** for a standard dense Transformer:

```text
params ≈ 12 × layers × hidden²   (plus embeddings: vocab × hidden, counted twice if not tied)
```

Llama-3.1-8B: 12 × 32 × 4096² ≈ 6.4B + embeddings ≈ 7.5–8B ✓. Use it to check model cards.

## Parameter count vs intelligence

- **Scaling laws**: performance improves predictably with more params + more data + more compute. That's why big labs race to scale.
- **But**: a well-trained 8B beats a poorly-trained 70B on many tasks. Post-training (SFT + RL) is where chat behavior, instruction following and reasoning come from.
- **Benchmarks are a proxy, not the truth** — they measure narrow tasks. For your use case (coding? chat? RAG?), test the actual model.
- **Diminishing returns per byte**: going 8B → 70B costs 8.75× the VRAM but isn't 8.75× smarter.

## Dense vs sparse (MoE)

| | Dense | Sparse / MoE |
|---|---|---|
| What runs per token | **all** parameters | only the **active** subset (e.g. 37B of 671B) |
| Examples | Llama-3.1-8B, Qwen3-8B | DeepSeek-V3 (671B/37B), Qwen3-235B-A22B (235B/22B), Qwen3-30B-A3B (30.5B/3.3B) |
| File size | = total params | = total params (all experts are stored!) |
| Decode speed | ∝ total params / bandwidth | ∝ **active** params / bandwidth |

**The "B" in the model name is the TOTAL count.** The active count is what determines speed. When you see "Qwen3-30B-A3B", read it as "30B stored, ~3B active per token". See [mixture-of-experts.md](mixture-of-experts.md) for the full picture.

## What parameter count means for local inference

1. **VRAM/RAM**: total params × bytes/weight must fit (weights + KV cache + overhead). See `02-hardware-and-vram/`.
2. **Decode speed**: roughly `memory bandwidth ÷ active bytes per token`. A 4B-active MoE model on a 4090 is fast; the same 30B dense model on the same card would be ~7× slower.
3. **MoE catch**: all 671B must be *loaded* even though only 37B are *used* per token. If the file doesn't fit in VRAM and you offload to CPU RAM, speed collapses (CPU bandwidth is 10–50× lower).
4. **Choosing a quant**: bigger models tolerate lower bits better (more redundancy). An 8B at Q4_K_M is nearly lossless; a 1B model at Q4 is noticeably degraded.

## My experiments

- (fill in: e.g. "benchmarked Q4_K_M vs Q8_0 on my card with llama-bench — decode speed identical, quality diff visible on X")
