# Context window

**Summary.** The context window is the maximum number of *tokens* the model can process at once — input + output combined. It's not a hard wall of "memory you can fill"; it's a budget that includes your system prompt, the whole conversation, any retrieved documents, **and** the answer being generated. What actually eats your VRAM is the **KV cache**, which grows linearly with the number of tokens in context — so context window and VRAM are the same decision from two sides.

## What fits in the window

```text
context budget (tokens)
├── system prompt / instructions
├── conversation history (all turns)
├── attached documents / RAG chunks
└── the model's own answer (must fit too!)
```

If the total exceeds the window, the runtime either errors, or (more often) **truncates** — usually dropping the oldest messages. Silent truncation is the #1 "the model forgot what I said" bug in local setups.

## Why the KV cache is the real cost

During generation the model recomputes attention over all previous tokens every step. Instead of recomputing, it stores each token's Key and Value vectors — the **KV cache**. Its size:

```text
KV cache bytes = 2 × layers × kv_heads × head_dim × seq_len × bytes_per_elem
```

The `2` is for K and V. See the full worked examples in [../cheatsheets/kv-cache-formula.md](../cheatsheets/kv-cache-formula.md).

Key consequences:

- **GQA/MLA matter**: Llama-3.1-8B has 8 KV heads (vs 32 query heads) → 4× smaller cache than a hypothetical MHA model. DeepSeek's MLA compresses K/V into a 512-dim latent → tiny cache, huge context.
- **Long context ≠ free**: doubling context doubles KV cache. A 128K context on an 8B model can cost more VRAM than the weights themselves.
- **Prefill cost**: processing a 32K-token prompt is one big matrix multiply (fast, compute-bound), but it *fills* the cache before you get a single output token.

## Context window vs "effective" context

- **Trained length**: how long the model was trained on. Qwen3 base: 32K; Qwen3 with YaRN: 131K. Llama-3.1: 128K.
- **Runtime limit**: what your VRAM allows. You can set `--ctx-size 32768` in llama.cpp even if the model supports 128K — or go over the trained length and accept quality degradation.
- **Quality decay**: models trained on 32K context often handle 8K fine, degrade past ~32K, and may hallucinate or lose coherence near the limit. "128K" in the name ≠ "128K of usable memory".
- **Rule of thumb**: run at or below the *trained* context length; only exceed it if you've tested quality on your task.

## Practical rules

1. **Budget your context**: system prompt + worst-case history + max answer ≤ window. Leave ~20% headroom.
2. **Watch the cache, not just the window**: before enabling 128K context on a 24 GB card, compute the KV cache (cheatsheet).
3. **Truncation is silent**: if answers get dumber mid-conversation, check whether old messages are being dropped.
4. **Summarize or compact**: for long-running chats, periodically summarize history into a short block instead of keeping every message.
5. **RAG beats long context for big corpora**: stuffing 200K tokens of docs into context is slower (prefill), more expensive (cache), and often *worse* than retrieving the 2–3 relevant chunks (topic 08).

## My experiments

- (fill in: e.g. "set ctx 32K vs 128K on Qwen3-8B — measured VRAM delta and quality on a needle-in-haystack test")
