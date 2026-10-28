# Model architecture basics (Transformer, enough for VRAM math)

**Summary.** Every modern LLM is a stack of identical Transformer *decoder* blocks. Each block = multi-head self-attention + a feed-forward network (FFN), wrapped in residual connections and normalization. The numbers that matter for local inference — **layers, hidden size, KV heads, head dim, vocab size** — are exactly the ones you need to compute weight size and KV cache size. Goal of this note: know what each part is and how big it is, not to derive attention math.

## The big picture

```text
text → tokens → embedding vectors (size = hidden)
     → [attention + FFN]  ×  N layers
     → final vector → logits (one score per vocab entry)
     → sample next token → append → repeat (autoregressive)
```

- **Embedding layer**: maps token id → vector of `hidden` floats. Size = `vocab × hidden`.
- **Output head**: maps final vector → `vocab` logits. Often the *same* matrix as the embedding (tied weights).
- **N layers** (a.k.a. depth): the repeated block. Llama-3.1-8B: 32. DeepSeek-V3: 61.

## Inside one block

### 1. Multi-head self-attention

For each token, the model builds three vectors by matrix-multiplying its embedding:

- **Query** (Q): "what am I looking for?"
- **Key** (K): "what do I contain?" (of every other token)
- **Value** (V): "what information do I offer?"

Attention score for token *t* looking at token *j*: `softmax(Q_t · K_j / √d)` — a weighted sum of all Values. Intuition: each token builds a new representation by *mixing in* information from the tokens it attends to (e.g. "it" attends to "animal").

**Multi-head**: this is done `H` times in parallel with different learned projections, each head with dimension `d_head = hidden / H`. Heads let different "subspaces" capture different relations (syntax, coreference, position…).

**The KV cache connection**: during generation, the K and V vectors of *all past tokens* are stored — that's the KV cache, and its size is `2 × layers × kv_heads × d_head × seq_len × bytes`.

### 2. Feed-forward network (FFN)

A 2-layer MLP applied independently to each token: expand `hidden → ~4×hidden` (or ~2.7× with SwiGLU), then back. **This is where ~2/3 of the parameters live** — and where MoE replaces the dense FFN with many small experts (see [mixture-of-experts.md](mixture-of-experts.md)).

### 3. Residuals + normalization

Each sub-layer's output is added back to its input (residual connection) and normalized (LayerNorm, or RMSNorm in Llama/Qwen/DeepSeek). Training stability, not something you tune.

## Variants you'll actually meet

| Variant | What it changes | Who uses it |
|---|---|---|
| **MHA** (multi-head attention) | Q, K, V all have H heads | original Transformer, GPT-2 |
| **MQA** (multi-query) | one shared K/V head | PaLM-era |
| **GQA** (grouped-query) | K/V have fewer heads than Q (e.g. 32 Q heads, 8 KV heads) | **Llama 3, Qwen, Mistral** — cuts KV cache 4× vs MHA |
| **RoPE** | rotary positional embeddings (position encoded as rotation of Q/K) | standard everywhere now |
| **RMSNorm** | cheaper normalization (no mean subtraction) | Llama, Qwen, DeepSeek |
| **SwiGLU** | gated FFN variant (3 matrices instead of 2) | Llama, Qwen, DeepSeek |
| **MLA** (multi-head latent attention) | K/V compressed into a small latent vector that's cached instead | **DeepSeek-V2/V3/R1** — tiny KV cache |

## Worked example: Llama-3.1-8B anatomy

From the model card: 32 layers, hidden 4096, 32 heads, **8 KV heads** (GQA), head_dim 128, vocab 128,256, FFN intermediate 14,336 (SwiGLU).

| Part | Math | Params |
|---|---|---|
| Embedding | 128,256 × 4,096 | 0.52B |
| Per layer — attention Q | 4,096 × 4,096 | 16.8M |
| Per layer — attention K,V | 2 × 4,096 × 1,024 | 8.4M |
| Per layer — attention O | 4,096 × 4,096 | 16.8M |
| Per layer — FFN (SwiGLU: 3 matrices) | 3 × 4,096 × 14,336 | 176M |
| Per layer total | | ≈ 218M |
| 32 layers | | ≈ 6.98B |
| + embedding + output head | | ≈ **8.0B** ✓ |

Takeaways: **FFN dominates the parameter count** (that's why MoE compresses it), and **attention K/V is small per layer but gets cached for every token** (that's why KV heads matter).

## DeepSeek-V3 anatomy (for contrast)

61 layers, hidden 7168, MLA (128 heads, head_dim 128, KV compressed to 512), first 3 layers dense FFN, remaining 58 layers MoE with **256 routed + 1 shared expert** (expert intermediate 2048), **8 routed experts active per token**, MTP depth 1. Total 671B, active 37B. (From the DeepSeek-V3 technical report, §2 & §4.2.)

## Why this matters for local inference

1. **Weight size** ≈ params × bytes/weight → pick quant + model to fit VRAM.
2. **KV cache** = `2 × layers × kv_heads × head_dim × ctx × bytes` → GQA/MLA models give you more context per GB (formula in [../cheatsheets/kv-cache-formula.md](../cheatsheets/kv-cache-formula.md)).
3. **MoE**: FFN replaced by experts → huge file, small active compute.
4. **Vocab size** affects embedding size and (slightly) per-token compute.

## Sources

- [The Illustrated Transformer — Jay Alammar](https://jalammar.github.io/illustrated-transformer/) (best visual intro)
- [Attention Is All You Need (2017)](https://arxiv.org/abs/1706.03762) (the paper)
- [DeepSeek-V3 Technical Report](https://arxiv.org/abs/2412.19437) (MLA + MoE + MTP in one model)
- Karpathy: *Let's build GPT* (video, see [../videos.md](../videos.md))

## My experiments

- (fill in: e.g. "printed Llama-3.1-8B config.json and verified the layer/head numbers above")
