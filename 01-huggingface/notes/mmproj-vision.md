# mmproj — the vision projector file

**Summary.** A vision model in the GGUF world is **two files**: the LLM GGUF and an **`mmproj`** GGUF containing the vision encoder (ViT tower) + projector. The mmproj encodes images into embeddings that get fed to the LLM (how it works: `07-multimodal-vision`). The mmproj is **model-specific** — it must match the exact base model — and its filename must start with `mmproj` for llama.cpp to auto-detect it. Ollama and LM Studio bundle it for you; llama.cpp needs it explicitly (or you use `-hf` and it downloads it).

## What's inside

- A **vision tower** (CLIP, SigLIP, or model-specific ViT) that turns image patches into embeddings.
- A **projector** (MLP) that maps those embeddings into the LLM's hidden space.
- Size ≈ the vision tower: a few hundred MB for 3–8B models (e.g. Qwen2.5-VL-7B `mmproj-F16.gguf` ≈ 0.6 GB), more for 32B+.
- Precision variants: `F16`, `BF16`, `F32` — and yes, **quantized mmprojs exist** (`Q8_0`), which save VRAM for little quality loss.

## Naming conventions (verified 2026-10-27)

| Naming | Seen in |
|---|---|
| `mmproj-model-f16.gguf` | older LLaVA-style repos |
| `mmproj-F16.gguf`, `mmproj-BF16.gguf`, `mmproj-F32.gguf` | `unsloth/Qwen2.5-VL-7B-Instruct-GGUF` |
| `mmproj-Qwen3VL-8B-Instruct-F16.gguf`, `mmproj-Qwen3VL-8B-Instruct-Q8_0.gguf` | official `Qwen/Qwen3-VL-8B-Instruct-GGUF` |

Rule: **filename must start with `mmproj`** (llama.cpp's auto-detection convention).

## Where to find them

1. **`ggml-org` pre-quantized multimodal collection** — every vision model llama.cpp supports, with matching mmproj, ready for `-hf`:
   `https://huggingface.co/collections/ggml-org/multimodal-ggufs-68244e01ff1f39e5bebeeedc`
2. **Community GGUF repos** — `unsloth`, `lmstudio-community`, `mradermacher`, official orgs (`Qwen`).
3. **Search recipe**: `https://huggingface.co/models?filter=image-text-to-text,gguf&sort=downloads`

## Supported vision models (llama.cpp, verified 2026-10-27)

Gemma 3 (4B/12B/27B) · Gemma 4 (E2B/E4B/26B-A4B/31B) · Qwen2-VL / Qwen2.5-VL (2B–72B) / Qwen3-VL · SmolVLM / SmolVLM2 (incl. video) · Pixtral 12B · Mistral Small 3.1 24B · InternVL 2.5 / 3 · Llama 4 Scout · Moondream2 · MiniCPM-V 4.x · LLaVA (legacy). Audio: Ultravox 0.5 and others. Full list: llama.cpp `docs/multimodal.md`.

## Using mmproj

**llama.cpp** (current flags, verified 2026-10-27):

```bash
# explicit files
llama-server -m Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf --mmproj mmproj-F16.gguf
llama-cli     -m model.gguf --mmproj mmproj.gguf -i          # interactive

# or let the server download model + mmproj from the Hub (mmproj auto-fetched)
llama-server -hf ggml-org/Qwen2.5-VL-7B-Instruct-GGUF:Q4_K_M

# options
llama-server -hf ggml-org/gemma-3-4b-it-GGUF --no-mmproj-offload   # keep tower on CPU
llama-server -hf <repo> --mmproj-device 0                          # pin tower to a GPU
llama-server -hf <repo> --no-mmproj                                # text-only mode
```

Short forms: `-mm` = `--mmproj`, `-mmu` = `--mmproj-url`. The tower is **offloaded to GPU by default** — that's extra VRAM on top of the LLM.

**Ollama**: the vision projector is baked into the model file — `ollama run qwen2.5vl:7b` and send an image, no separate file.

**LM Studio**: downloads the mmproj automatically with the model.

**vLLM / Transformers**: no mmproj at all — they use the native vision tower from the safetensors repo.

## Pitfalls

- **Version mismatch**: an mmproj from Qwen2.5-VL-7B does not work with Qwen3-VL-8B (different tower architecture). Match base model *and* revision.
- **VRAM**: tower + embeddings add real memory; long image sequences (many tiles) can surprise you. `--no-mmproj-offload` moves the tower to CPU (slower image encoding, less VRAM).
- **Image cost in context**: each image consumes dozens-to-hundreds of "tokens" — budget your context accordingly (see `07-multimodal-vision`).
- **OCR models** (PaddleOCR-VL, GLM-OCR, DeepSeek-OCR, …) need their specific prompt structure — check the linked llama.cpp discussions, not the generic chat template.

## My experiments

- (fill in: e.g. "Qwen2.5-VL-7B Q4_K_M + mmproj-F16 on 16 GB: 5.0 GB LLM + 0.6 GB tower + ctx 8k fits; 4096px image ≈ 1.2k image tokens")
