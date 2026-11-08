# VRAM for vision models — why they need more headroom

**Summary.** A vision model needs more VRAM than its text twin for three reasons: (1) the **vision encoder weights** (mmproj — a few hundred MB to ~1 GB, on top of the LLM), (2) **visual tokens in the KV cache** — one image is 1,600–6,400 tokens, each cached like any token, (3) **encoder activation memory** during the ViT forward pass (scales with pixel count). The fixes: quantize the mmproj, offload it to CPU (`--mmproj-device none`), cap image token budgets, and keep `--ctx-size` honest. Base math: [02-hardware-and-vram/cheatsheets/vram-budget.md](../../02-hardware-and-vram/cheatsheets/vram-budget.md).

## The three extra costs

### 1. Encoder weights (mmproj)

| Model | mmproj size (F16/BF16) |
|---|---|
| Qwen2.5-VL-7B | ~0.6 GB |
| Llama 3.2 11B | ~0.6 GB |
| Gemma 3 12B | ~0.8 GB |
| Qwen2.5-VL-72B | ~1.5 GB+ |

Q8_0 mmproj ≈ half the size; F32 ≈ double. This is a **fixed cost** — it's in VRAM as soon as the model loads, even for text-only requests.

### 2. Visual tokens in the KV cache

Image tokens are cached like text tokens. Per-token KV for an 8B-class model ≈ 128 KB (fp16):

| Image | Tokens | KV @ 8B fp16 | KV @ 8B q4_0 |
|---|---|---|---|
| Qwen VL 1080p | ~2,645 | ~0.34 GB | ~0.08 GB |
| Llama 3.2 4-tile | 6,400 | ~0.82 GB | ~0.20 GB |
| 4 images × 2,645 | 10,580 | ~1.35 GB | ~0.34 GB |

Llama 3.2 (Mllama) adds a **cross-attention KV** for image tokens on top of the self-attention cache — the known memory hotspot that makes the 90B vision model so hungry.

### 3. Encoder activations

The ViT forward pass over all patches holds intermediate tensors in VRAM — this scales with **pixel count, not token count**. It's transient (freed after encoding) but it's why a 4K image can OOM a setup that handles 1080p fine. This is what `--mtmd-batch-max-tokens` (llama.cpp) caps: how many image tokens are encoded per batch.

## Budget formula

```text
vision VRAM ≈ LLM weights + mmproj + KV(ctx + image tokens) + encoder activations + 1 GB
```

Decision rules:

- **Fits text but not vision?** → `--mmproj-device none` (encoder to CPU; slower prefill, VRAM saved), quantize the mmproj to Q8_0, cap image tokens.
- **Multi-image workloads** → budget N × image tokens into `--ctx-size`, or use `--cache-type-k/v q4_0` to quarter the KV cost.
- **Video** → frames × tokens/frame: a 30 s clip @ 1 fps of 1080p frames ≈ 30 × 2,645 ≈ 79k tokens on Qwen VL — that's a 128k-context problem, not a 32k one.
- **The text baseline still applies** — vision headroom is *on top of* the weights+KV budget from [02-hardware-and-vram](../../02-hardware-and-vram/cheatsheets/vram-budget.md). A "comfortable" 16 GB text setup can be a "tight" vision setup.

## Sources

- [This repo — 02-hardware-and-vram/cheatsheets/vram-budget.md (base math)](../../02-hardware-and-vram/cheatsheets/vram-budget.md)
- [This repo — 07 notes/how-vision-llms-work.md (token counts per family)](how-vision-llms-work.md)
- [Efficient LLaMA-3.2-Vision (measured visual token counts)](https://arxiv.org/html/2504.00557v1)
- [llama.cpp — mtmd README (`--mtmd-batch-max-tokens`, `--mmproj-device`)](https://github.com/ggml-org/llama.cpp/blob/master/tools/mtmd/README.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B Q4_K_XL + mmproj-BF16: text-only load 21.4 GB; with a 1080p image in context, nvidia-smi peaks at 22.1 GB (the ~0.7 GB = mmproj + 2,645 image tokens @ q4_0 KV); `--mmproj-device none` dropped the peak to 21.6 GB at the cost of +0.6 s TTFT")
