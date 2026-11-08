# Index — 00-foundations

**How to read this folder.** This is the conceptual base everything else builds on — read the `notes/` **in the numbered order below**, each one assumes the previous. Cheatsheets are for use *while working* (counting tokens, picking a quant, estimating VRAM), not for first reading. `docs/` holds the primary sources for every note; `videos.md` is optional.

## Start here

`notes/what-is-a-token.md` — every other topic in this repo (context, VRAM, speed) is expressed in tokens.

## Reading order

1. `notes/what-is-a-token.md` — what a token is, how to count them
2. `notes/model-architecture-basics.md` — the Transformer, at exactly the level needed for VRAM math
3. `notes/what-are-parameters.md` — what "7B" / "27B" actually means
4. `notes/model-formats-and-quantization.md` — GGUF, quants, why 4-bit exists
5. `notes/context-window.md` — KV cache and why context eats VRAM
6. `notes/inference-pipeline.md` — prefill vs decode, where your time actually goes
7. `notes/mixture-of-experts.md` — MoE, why "30B-A3B" behaves like a small model *(skip if you only run dense models)*
8. `notes/multi-token-prediction.md` — MTP, the idea behind speculative decoding *(skip until you meet it in [04-llama-cpp](../04-llama-cpp/))*
9. `notes/system-prompts-and-chat-templates.md` — how your message becomes a prompt *(bridge into [06-sampling-and-prompting](../06-sampling-and-prompting/))*

## Quick reference (cheatsheets)

- `cheatsheets/tokenizer-commands.md` — count/inspect tokens for a model
- `cheatsheets/quantization-levels.md` — what Q4_K_M etc. mean, size/quality tradeoffs
- `cheatsheets/kv-cache-formula.md` — estimate KV cache size for a given context

## Sources & videos

- `docs/` — one references file per note
- `videos.md` — curated videos
