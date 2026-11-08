# How vision LLMs work — image → patches → visual tokens

**Summary.** A vision LLM is a text LLM plus a **vision encoder** (a ViT trained on image-text pairs, e.g. CLIP/SigLIP) and a **projector** (a small MLP or cross-attention adapter) that maps the encoder's patch embeddings into the LLM's embedding space. The image becomes a *sequence of visual tokens* that is prepended to the text tokens — from the LLM's point of view, an image is just a long, dense paragraph it was trained to read. The practical consequence: **each image costs hundreds to thousands of context tokens**, and that cost is a dial you can turn (resolution, tiling, token budget). One-page version: `cheatsheets/image-token-cost.md`.

## The pipeline

```text
image (H×W px)
  → resize / tile to model's input shape
  → split into patches (14×14 or 28×28 px)
  → ViT encoder (CLIP / SigLIP / model-specific) → one embedding per patch
  → (optional) spatial merge (2×2 → ¼ the tokens)
  → projector (MLP / cross-attention) → LLM hidden space
  → N visual tokens, spliced into the prompt where the <image> placeholder is
  → LLM runs as usual (prefill over text + visual tokens, then decode)
```

Key facts:

- The encoder is **frozen or lightly tuned** in most open models — the LLM learns to *read* the visual tokens, it doesn't see pixels.
- Visual tokens go through **full (bidirectional) attention among themselves** in the encoder, then **causal attention** in the LLM — the model isn't generating them, just consuming them.
- **Video = sampled frames**: a video is N frames, each encoded like an image (Qwen VL uses a 3D/strided approach to share work across frames). Cost scales with frame count.
- In the GGUF world the encoder + projector live in a separate **mmproj** file — see [01-huggingface/notes/mmproj-vision.md](../../01-huggingface/notes/mmproj-vision.md).

## How many tokens does an image cost?

This is the number that matters for context and VRAM. It varies by model family:

| Family | Scheme | Tokens per image |
|---|---|---|
| **Qwen2-VL / 2.5-VL / 3-VL** | dynamic: 1 token per 28×28 px, smart-resized | **H×W / 784**, clamped by `min_pixels`/`max_pixels` (default range 4–16,384; recommended 256–1,280) |
| **Llama 3.2 (Mllama)** | fixed tiles: 560×560 px → 1,600 tokens, max 4 tiles | **1,600** (small img) up to **6,400** (≤1120×1120) |
| **Gemma 3** | fixed 896×896 crop → 256 tokens; Pan & Scan tiles big images | **256 per crop** (a 2×2-tiled photo = 1,024) |
| **Gemma 4** | selectable token budget | **70 / 140 / 280 / 560 / 1,120** per image |
| **MiniCPM-V / SmolVLM / Moondream** | small dynamic/fixed encoders | a few hundred (budget-friendly by design) |
| **Pixtral / InternVL** | dynamic tiling | ~1,000+ for high-res |

Worked examples (Qwen2.5-VL, 28×28 px/token):

| Image | Pixels | Tokens (H×W/784) |
|---|---|---|
| 672×672 thumbnail | 451,584 | ~576 |
| 1024×1024 screenshot | 1,048,576 | ~1,338 |
| 1920×1080 photo | 2,073,600 | ~2,645 |
| 3840×2160 (4K) | 8,294,400 | ~10,579 |

Llama 3.2: a 384×384 icon → 1,601 tokens (1 tile + 1 placeholder); a 720p screenshot → ~6,404 (4 tiles). **Downscale before sending** — a 4K screenshot into Llama 3.2 costs 6,400 tokens whether or not you need the detail.

## Impact on context, KV cache, and speed

1. **Context**: a 32k context that fits ~30k words of text fits only ~27k words + one 4K Qwen-VL image. Multi-image RAG (screenshots, charts) eats context fast — this is the #1 surprise for local vision users.
2. **KV cache**: visual tokens are cached like any token (per-token cost from [02-hardware-and-vram](../../02-hardware-and-vram/cheatsheets/vram-budget.md)). Llama 3.2's cross-attention layers add *extra* KV for image tokens on top of the self-attention cache — a known memory hotspot.
3. **Prefill**: image encoding (the ViT forward pass) + prefill over all visual tokens is where TTFT goes. A 2,600-token image on a 7B model can add 1–3 s of prefill on a good GPU.
4. **Decode is unaffected per token** — but longer context = slower attention as the sequence grows.
5. **OCR quality is a token-budget problem**: fine text needs more visual tokens. A document at 576 tokens loses small print; at 2,600+ it's readable. If OCR fails, raise the resolution/token budget before blaming the model.

## The dials you can turn

| Dial | Where | Effect |
|---|---|---|
| `min_pixels` / `max_pixels` | Qwen VL processor / llama.cpp `--image-min-tokens` / `--image-max-tokens` | clamp the token range |
| image resize (client-side) | your code / `ffmpeg` for video | cheapest lever — downscale before sending |
| Pan & Scan on/off | Gemma 3 (inference-time tiling) | quality vs tokens |
| token budget (70–1,120) | Gemma 4 | explicit quality/cost choice |
| `--video-fps` | llama.cpp | frames sampled from video |
| `--mtmd-batch-max-tokens` | llama.cpp | image tokens per encode batch (VRAM) |

Rule of thumb: **send the image at the resolution the task needs, not the resolution the camera took it at.** A UI screenshot at 1080p is plenty; a 4K photo of a whiteboard is not.

## Sources

- [HF — Vision language models (blog: LLaVA architecture, projector)](https://huggingface.co/blog/vlms)
- [Cameron Wolfe — Vision LLMs deep dive (encoder/projector/LLM)](https://cameronrwolfe.substack.com/p/vision-llms)
- [Qwen2-VL blog (dynamic resolution, min/max pixels)](https://qwenlm.github.io/blog/qwen2-vl)
- [Gemma 3 technical report (SigLIP 896×896, 256 tokens, Pan & Scan)](https://arxiv.org/html/2503.19786v1)
- [Google — Gemma vision (token budgets 70–1,120)](https://ai.google.dev/gemma/docs/capabilities/vision)
- [Efficient LLaMA-3.2-Vision (measured visual token counts: 1,601–6,404)](https://arxiv.org/html/2504.00557v1)
- [Fireworks — Qwen2.5-VL tokens-per-image table](https://docs.fireworks.ai/faq-new/billing-pricing/how-many-tokens-per-image)
- [This repo — 01-huggingface/notes/mmproj-vision.md](../../01-huggingface/notes/mmproj-vision.md)

## My experiments

- (fill in: e.g. "Qwen3-VL-8B: 1080p screenshot ≈ 2.6k image tokens → TTFT 1.9s vs 0.3s text-only; same doc at 672px (576 tokens) still read the table but missed the footnote; Llama 3.2-11B on the same 1080p image: 6,404 tokens — 2.4× the Qwen cost")
