# Cheatsheet — vision models

**TL;DR.** Quality leader: **Qwen VL** (dynamic resolution, best OCR, video). Efficient all-rounder: **Gemma 3/4**. Meta line: **Llama 3.2 Vision / Llama 4 Scout**. Small tier: **MiniCPM-V / SmolVLM / Moondream**. Classic: **LLaVA** (now outclassed at equal size). Every open vision model needs a matching **mmproj** file in the GGUF world. Deep dive: `notes/vision-models.md`.

## The table

| Model | Sizes (local-relevant) | Tokens/image | Min VRAM (Q4) | Best at |
|---|---|---|---|---|
| Qwen2.5-VL | 3B, 7B, 32B, 72B | H×W/784 (dynamic, ~576–2,645 typical) | 7B ≈ 8–10 GB | OCR, documents, video |
| Qwen3-VL | 2B, 4B, 8B, 30B-A3B, 32B | dynamic | 8B ≈ 8–10 GB · 30B-A3B ≈ 18 GB | OCR, grounding, video (2026 leader) |
| Gemma 3 | 4B, 12B, 27B | 256 per 896² crop (Pan & Scan tiles) | 4B ≈ 4–6 GB | laptops, multilingual |
| Gemma 4 | E2B, E4B, 26B-A4B, 31B | 70–1,120 selectable | 4B ≈ 4–6 GB | efficiency, vision built in |
| Llama 3.2 Vision | 11B, 90B | 1,600/tile, max 6,400 | 11B ≈ 10–12 GB | documents, 128k ctx |
| Llama 4 Scout | 17B active / 109B MoE | tiling (new) | ≈ 24–32 GB (Q4) | frontier-class multimodal MoE |
| MiniCPM-V | 8B (2.6 / 4.x) | dynamic | ≈ 6–8 GB | small-but-strong, video |
| SmolVLM | 250M–2.2B | ~256 | 2–4 GB | phones / 8 GB |
| Moondream | ~1.9B | ~256 | 2–4 GB | ultra-fast |
| Pixtral | 12B | ~1,000+ | ≈ 10 GB | multi-image, charts, Apache-2.0 |
| InternVL | 8B–78B | dynamic tiling | 8B ≈ 8 GB | benchmark scores |
| LLaVA 1.5/1.6 | 7B, 13B | ~576 | ≈ 6–8 GB | the classic recipe (teaching) |

## Ollama tags

`qwen2.5vl:7b` · `qwen3-vl:8b` · `gemma3:4b` / `gemma3:12b` · `llama3.2-vision:11b` · `minicpm-v` · `llava:13b` · `moondream` · `pixtral:12b`

## Pick one

- **8 GB** → Gemma 3 4B (or Moondream for speed)
- **16 GB** → Qwen2.5-VL-7B (OCR) or Llama 3.2 11B (documents)
- **24 GB** → Qwen3-VL-8B (dense/fast) or Qwen3-VL-30B-A3B (MoE)
- **48 GB** → Llama 4 Scout Q4
- **Video** → Qwen VL family
- **Multi-image / charts** → Pixtral or Qwen VL

Verified against HF model cards / official blogs — 2026-11-08.
