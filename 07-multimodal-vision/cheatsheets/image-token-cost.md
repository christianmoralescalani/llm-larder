# Cheatsheet — image token cost

**TL;DR.** An image is N context tokens; N depends on the family. Qwen VL: **H×W / 784** (1 token per 28×28 px, clamped by min/max pixels). Llama 3.2: **1,600 per 560×560 tile, max 4 tiles (6,400)**. Gemma 3: **256 per 896×896 crop** (Pan & Scan tiles). Gemma 4: **70–1,120 selectable**. Downscale before sending; budget N against `--ctx-size` and the KV cache. Deep dive: `notes/how-vision-llms-work.md`.

## Quick calc

| Image | Qwen VL (H×W/784) | Llama 3.2 (tiles) | Gemma 3 (crops) |
|---|---|---|---|
| 384×384 icon | ~189 | 1,600 (1 tile) | 256 |
| 672×672 | ~576 | 1,600 | 256 |
| 1024×1024 | ~1,338 | 3,200 (2 tiles) | ~512 (2×2 P&S) |
| 1920×1080 | ~2,645 | 6,400 (4 tiles) | ~1,024 (2×3 P&S) |
| 3840×2160 (4K) | ~10,579 | 6,400 (capped) | ~2,048 (4×3 P&S) |

## KV cost (8B-class, fp16 ≈ 128 KB/token)

tokens × 128 KB: 576 ≈ 0.07 GB · 1,600 ≈ 0.20 GB · 2,645 ≈ 0.34 GB · 6,400 ≈ 0.82 GB. With q4_0 KV: ÷4.

## The dials

| Dial | Where | Effect |
|---|---|---|
| client-side resize | your code / `ffmpeg` | cheapest — fewer pixels in, fewer tokens out |
| `min_pixels` / `max_pixels` | Qwen VL processor / llama.cpp `--image-min-tokens` / `--image-max-tokens` | clamp the token range |
| `--mtmd-batch-max-tokens` | llama.cpp | encoder batch size (VRAM) |
| `--mm-processor-kwargs '{"max_pixels": N}'` | vLLM | cap Qwen VL pixels |
| `--video-fps` | llama.cpp | frames sampled from video |
| token budget 70–1,120 | Gemma 4 | explicit quality/cost choice |

## Rules

1. Send the resolution the **task** needs, not the camera's.
2. Budget image tokens into `--ctx-size` (multi-image = N × tokens).
3. OCR failing → raise the token budget before blaming the model.
4. Video = frames × tokens/frame (10 s @ 1 fps, 1080p ≈ 26k tokens on Qwen VL).

Verified against official model docs (Qwen2-VL blog, Gemma 3 report, Llama 3.2 measurements) — 2026-11-08.
