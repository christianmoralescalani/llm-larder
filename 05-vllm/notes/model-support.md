# Model support — what vLLM can load

**Summary.** vLLM runs **Hugging Face weights** (safetensors) natively — no conversion. Coverage: LLMs (dense + MoE), vision-language (Qwen2.5-VL, LLaVA, Pixtral, …), audio (Whisper, Qwen2-Audio), embeddings/rerankers, and a long list of quant formats (AWQ, GPTQ, FP8, GGUF, bitsandbytes, …). If a model isn't in vLLM's native list, it can still run through the **Transformers modeling backend** fallback. Your local GGUF files (from 04-llama-cpp) load directly — `--model /path/to/model.gguf` or a folder containing them.

## Native architectures

The full list is long and moving; the shape of it:

| Family | Examples | Notes |
|---|---|---|
| Dense LLMs | Llama 3.x/4, Qwen2/2.5/3, Mistral, Gemma 3, Phi-4, GPT-OSS, OLMo, DeepSeek-V3/R1 | the bread and butter |
| MoE | DeepSeek-V3/R1, Mixtral, Qwen3-MoE, GPT-OSS, GLM | `--enable-expert-parallel` for multi-GPU expert sharding |
| Mamba/hybrid | Jamba, Falcon-Mamba, Nemotron-H, Qwen3-Next | SSM state cache instead of (or alongside) KV |
| Vision-language | Qwen2-VL / Qwen2.5-VL / Qwen3-VL, LLaVA (incl. NeXT), LLaVA-OneVision, Pixtral, Idefics3, MiniCPM-V, InternVL, GLM-4V, Kosmos-2 | send images as base64 or URLs in `messages[].content` |
| Audio | Whisper (transcription), Qwen2-Audio, Kimi-Audio, Step-Audio | `--task` or model auto-detect; needs `vllm[audio]` deps (pip) / custom image (Docker) |
| Embeddings | BGE, E5, GTE, all-MiniLM, jina-embeddings, NV-Embed, SFR | `task="embed"` |
| Rerankers | bge-reranker, jina-reranker, rank-T5 | `task="score"` |
| Reward models | Llama-2-13B-reward, DeepSeek-RM | `task="reward"` |
| Multimodal gen | (not the focus) | image gen is out of scope here |

Check the live matrix: [Supported Models](https://docs.vllm.ai/en/stable/models/supported_models/) (auto-generated from `vllm/model_executor/models`).

## Vision models in practice

```bash
# serve a VLM (Docker)
docker run --rm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-VL-7B-Instruct \
  --limit-mm-per-prompt '{"image": 4}'
```

```bash
# image in a chat request: base64 data URL or public URL
curl http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "Qwen/Qwen2.5-VL-7B-Instruct",
  "messages": [{"role": "user", "content": [
    {"type": "image_url", "image_url": {"url": "https://example.com/cat.jpg"}},
    {"type": "text", "text": "What is in this image?"}
  ]}]}'
```

- `--limit-mm-per-prompt` caps items per prompt per modality (default 999 each); also `num_frames`, `width`, `height` options for video.
- `--allowed-local-media-path` / `--allowed-media-domains` control where media can be fetched from.
- Images cost **context tokens** (an image ≈ hundreds–thousands of tokens depending on size) — budget `--max-model-len` accordingly.
- Video: `{"type": "video_url", ...}` for models that support it (Qwen2.5-VL, LLaVA-OneVision, …).

## Audio

- **Whisper**: `vllm serve openai/whisper-large-v3 --task transcription` (or `translation`); OpenAI-compatible `/v1/audio/transcriptions`.
- **Qwen2-Audio / Kimi-Audio**: chat with audio in `messages` (`input_audio` content part).
- The Docker image does **not** ship the optional audio deps — for audio models build a custom image (`RUN uv pip install --system vllm[audio]==<version>`), see `notes/docker.md`.

## Quant formats

| Format | Flag / how | Typical use |
|---|---|---|
| **AutoAWQ** (INT4/INT8) | auto from repo, or `-q awq` | 4-bit weights, FP16 activations; widely available on HF |
| **GPTQModel** (INT4/INT8) | auto, or `-q gptq` | classic 4-bit; older GPTQ repos need `gptq` (legacy) vs `gptqmodel` |
| **FP8** (W8A8) | auto, or `-q fp8` | Hopper/Ada datacenter GPUs; near-lossless, ~2× VRAM savings |
| **NVFP4 / MXFP4** (W4A4) | auto, or `-q modelopt` / `-q mxfp4` | Blackwell (B200/GB200) 4-bit |
| **GGUF** | auto, or `-q gguf` | **your llama.cpp quants load directly** (Q4_K_M etc.); no conversion |
| **bitsandbytes** (INT8/INT4) | `-q bitsandbytes` | 8-bit/4-bit, easy but slower kernels |
| **Intel Neural Compressor** (INT4/INT8) | auto | INC checkpoints |
| **LLM Compressor** | auto | FP8 W8A8, INT8 W8A8, INT4 W4A16 checkpoints (the recommended tool to *create* them) |
| **compressed-tensors** | auto | HF compressed-tensors format |
| **torchao** | `-q torchao` | int8/int4/fp8 via torchao |
| **GPTQ (exllama/v100)** | legacy | old checkpoints only |

Rules of thumb:

- **Most HF quant repos need no flag** — `quantization_config` in the repo drives it.
- **FP8 for Hopper+**, **AWQ/GPTQ 4-bit for consumer cards** (RTX 30/40/50), **GGUF** if you already quantized in 04-llama-cpp.
- `--dtype` pairs with the quant (AWQ → `half`; FP8 → `auto`/`bfloat16`).
- KV cache quantization is separate: `--kv-cache-dtype fp8` (halves KV VRAM) regardless of weights.

## Transformers fallback

Models without a native vLLM implementation can run via the **Transformers modeling backend** (`--model-impl transformers`): slower (no vLLM kernels), but broad coverage for new/exotic architectures while vLLM catches up.

## Gotchas

1. **HF weights, not GGUF-only repos** — a repo that only has GGUF files still works (`-q gguf`), but the standard path is safetensors.
2. **New model, old vLLM** — if your model isn't supported yet: pin a newer image tag, or `--model-impl transformers`, or build from `main`.
3. **Vision needs VRAM headroom** — image tokens eat context; a 7B VLM with 4 large images can need more KV than the same 7B text-only.
4. **Audio = custom image** — optional deps are excluded from the official Docker image for licensing.
5. **MoE + quant** — some quants have limited MoE support; check the model's HF card for a vLLM-tested quant.
6. **`--trust-remote-code`** — some repos ship custom code; vLLM won't run it unless you pass the flag (security: only for repos you trust).

## Sources

- [Supported models](https://docs.vllm.ai/en/stable/models/supported_models/)
- [Quantization](https://docs.vllm.ai/en/stable/features/quantization/)
- [Multimodal inputs](https://docs.vllm.ai/en/stable/features/multimodal_inputs/)
- [Speech-to-text serving (Whisper)](https://docs.vllm.ai/en/stable/serving/online_serving/speech_to_text/)
- [Pooling models](https://docs.vllm.ai/en/stable/models/pooling_models/)

## My experiments

- (fill in: e.g. "Qwen3.8-27B GGUF (Q4_K_M, from 04-llama-cpp) served in vLLM with `-q gguf`: X t/s decode vs llama.cpp Y t/s — same weights, different engine; Qwen2.5-VL-7B with 4 images: Z tokens of context used")
