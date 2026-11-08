# Vision model landscape — who's who, and when to pick which

**Summary.** The open vision-LLM field in 2026: **Qwen VL** is the quality leader (dynamic resolution, best OCR, video), **Gemma 3/4** is the efficient all-rounder (small token budget, runs anywhere), **Llama 3.2/4** is the Meta line (strong documents, fixed tiling), **MiniCPM-V / SmolVLM / Moondream** are the small-model tier (laptops, 8 GB), and **Pixtral / InternVL** round it out. By 2026 vision is also *built into* the mainstream text families (Qwen3.5 accepts images at every size, Gemma 4 reads images at every size, Ministral 3 ships a vision encoder) — so "text model + vision" is becoming one download. Full table: `cheatsheets/vision-models.md`.

## The families

### Qwen VL (Qwen2-VL → 2.5-VL → 3-VL)

- **Sizes**: 2B–72B (2.5-VL); Qwen3-VL: 2B, 4B, 8B, 30B-A3B, 32B, 235B-A22B.
- **Strengths**: best-in-class OCR/document reading among open models; dynamic resolution (you control token cost — H×W/784); video understanding (3D patching, 1h+ clips); visual grounding (bounding boxes as JSON); code-from-image (diagrams, HTML/CSS).
- **Weaknesses**: the big ones are big (72B/235B need multi-GPU); dynamic resolution means token cost varies — budget for it.
- **Local sweet spot**: **Qwen2.5-VL-7B** (16 GB) and **Qwen3-VL-8B / 30B-A3B** (the MoE runs fast on 24 GB).

### Gemma 3 / Gemma 4 (Google)

- **Sizes**: Gemma 3: 4B, 12B, 27B (1B is text-only). Gemma 4: E2B, E4B, 26B-A4B, 31B.
- **Strengths**: very efficient — 256 tokens per 896×896 crop (Gemma 3), selectable 70–1,120 token budgets (Gemma 4); strong multilingual; document/handwriting OCR; permissive Gemma license.
- **Weaknesses**: fixed-resolution crops (Pan & Scan tiles big images — more tokens than it looks); OCR of tiny print trails Qwen VL.
- **Local sweet spot**: **Gemma 3 4B** (8–12 GB, the default "I need vision on a laptop" pick) and **12B** (24 GB).

### Llama 3.2 / Llama 4 (Meta)

- **Sizes**: Llama 3.2 Vision 11B, 90B (1B/3B are text-only); Llama 4 Scout (17B active / 109B total MoE) and Maverick (17B active / 400B total) are natively multimodal (image + video).
- **Strengths**: strong documents/charts; 128k context; Llama 4 Scout is the first frontier-class open multimodal MoE that fits on 2×24 GB (Q4).
- **Weaknesses**: fixed 560×560 tiling = 1,600 tokens per tile (expensive for small images — a 384px icon still costs 1,600); Llama 3.2 90B is heavy; Llama 4 tooling is still catching up.
- **Local sweet spot**: **Llama 3.2 11B** (16–24 GB) for documents; **Llama 4 Scout Q4** (48 GB) if you want frontier-class locally.

### MiniCPM-V / SmolVLM / Moondream (the small tier)

- **MiniCPM-V 2.6 / 4.x** (OpenBMB): 8B dense, beats much bigger models on document/video benchmarks; the "laptop power user" pick.
- **SmolVLM** (Hugging Face): 250M–2.2B, runs on phones/8 GB; quality is basic but it's *there*.
- **Moondream** (~1.9B): ultra-fast on 8 GB; weak at fine detail.
- This tier exists so vision works on the same hardware as 7B text models.

### Pixtral / InternVL / LLaVA (the rest)

- **Pixtral** (Mistral): 12B, strong OCR + multi-image, Apache-2.0.
- **InternVL** (Shanghai AI Lab): up to 78B, top benchmark scores, heavier.
- **LLaVA 1.5/1.6** (the classic): 7B/13B — historically important (invented the recipe: ViT + MLP + LLM), but now outclassed by Qwen VL/Gemma 3 at the same size. Still the best *teaching* example of the architecture.

## Choosing

| Your situation | Pick |
|---|---|
| 8 GB laptop, "can it see?" | Gemma 3 4B, or Moondream/SmolVLM for speed |
| 16 GB, documents + OCR | Qwen2.5-VL-7B or Llama 3.2 11B |
| 24 GB, best quality you can run | Qwen3-VL-8B (dense, fast) or Qwen2.5-VL-32B (Q4, slow) |
| 24 GB, MoE speed + vision | Qwen3-VL-30B-A3B |
| 48 GB, frontier-class locally | Llama 4 Scout Q4 |
| Video understanding | Qwen2.5-VL / Qwen3-VL (3D patching) |
| Multi-image / charts | Pixtral or Qwen VL |

## Cross-cutting notes

- **GGUF availability**: all of the above have community GGUFs + matching mmproj on HF (unsloth, QuantFactory, bartowski, …). The mmproj must match the exact base model — see [01-huggingface/notes/mmproj-vision.md](../../01-huggingface/notes/mmproj-vision.md).
- **Vision ≠ text skill**: a 7B vision model is a 7B text model with eyes — vision doesn't fix weak instruction-following. Pick the text model first, then check it has vision.
- **Token cost is a first-class spec** — compare models on tokens/image, not just params (see `cheatsheets/image-token-cost.md`).

## Sources

- [Qwen2-VL blog (dynamic resolution, min/max pixels)](https://qwenlm.github.io/blog/qwen2-vl)
- [Qwen2.5-VL blog (video, grounding)](https://qwenlm.github.io/blog/qwen2.5-vl)
- [Gemma 3 technical report (SigLIP 896×896, 256 tokens, Pan & Scan)](https://arxiv.org/html/2503.19786v1)
- [Google — Gemma vision (token budgets 70–1,120)](https://ai.google.dev/gemma/docs/capabilities/vision)
- [Meta — Llama 3.2 (vision models, 128k context)](https://ai.meta.com/blog/llama-3-2-connect-2024-vision-edge-mobile-devices)
- [Meta — Llama 4 (natively multimodal)](https://ai.meta.com/blog/meta-llama-4-multimodal-intelligence)
- [MiniCPM-V (OpenBMB)](https://github.com/OpenBMB/MiniCPM-V)
- [SmolVLM (Hugging Face blog)](https://huggingface.co/blog/smolvlm)
- [Moondream](https://github.com/vikhyat/moondream)
- [Pixtral (Mistral)](https://mistral.ai/news/pixtral-large)
- [InternVL](https://github.com/OpenGVLab/InternVL)

## My experiments

- (fill in: e.g. "Qwen3-VL-8B vs Gemma 3 12B on my 30-receipt OCR set: Qwen 96% field accuracy vs Gemma 88%; Gemma answered 2.3× faster on the same 4090; Llama 3.2 11B read the same 1080p screenshot at 6,400 tokens vs Qwen's 2,645 — the token-cost gap shows up directly in TTFT")
