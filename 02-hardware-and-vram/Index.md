# Index — 02-hardware-and-vram

**How to read this folder.** This is the *math* folder — it answers "will it fit, and how fast?" Read the notes in order: formula first, then the three things that plug into it (quant, KV cache, offload), then hardware choice. The single cheatsheet is the one you'll open most often — "I have X GB, what fits?"

## Start here

[notes/vram-formula.md](notes/vram-formula.md) — the whole topic in one equation: weights + KV cache + overhead.

## Reading order

1. [notes/vram-formula.md](notes/vram-formula.md) — the master formula, worked by model size
2. [notes/quantization-math.md](notes/quantization-math.md) — bytes per weight: how the quant changes the first term
3. [notes/kv-cache-math.md](notes/kv-cache-math.md) — how context length and batch size eat the second term
4. [notes/vram-vs-ram-cpu.md](notes/vram-vs-ram-cpu.md) — what actually happens when it doesn't fit (speed cliff, CPU offload)
5. [notes/gpu-guide.md](notes/gpu-guide.md) — consumer cards: what VRAM you can buy, what's worth it
6. [notes/multi-gpu.md](notes/multi-gpu.md) — splitting a model across cards *(only if one card can't hold it)*
7. [notes/apple-silicon.md](notes/apple-silicon.md) — unified memory math *(only if you're on a Mac)*

## Quick reference (cheatsheets)

- [cheatsheets/vram-budget.md](cheatsheets/vram-budget.md) — "I have X GB → which models/quants fit" — the table to keep open while shopping

## Sources & videos

- [docs/references.md](docs/references.md) — official specs & docs
- [videos.md](videos.md) — curated videos
