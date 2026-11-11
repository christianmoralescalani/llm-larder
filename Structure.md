# LocalLLM — Repository Structure

> Knowledge base for running LLMs locally: notes, videos, slides (diapos), cheatsheets, and docs.
> This file is the **single source of truth** for how the repository is organized. Update it whenever a topic is added, split, or renamed.

---

## 1. Content types

Every topic folder can use the same 5 content types, so navigation is always predictable. Subfolders are created when the first file is added — empty folders are not kept:

| Folder / file       | What goes there                                                        | Format            |
|---------------------|------------------------------------------------------------------------|-------------------|
| `notes/`            | Long-form written notes, explanations, step-by-step guides            | `.md`             |
| `cheatsheets/`      | One-page references: commands, flags, formulas, tables                | `.md`             |
| `slides/`           | Slide decks (diapos) for presentations                                | `.pptx` / `.pdf` / `.md` |
| `docs/`             | External docs, official references, saved PDFs, API specs             | links / `.pdf`    |
| `videos.md`         | Curated video list (YouTube etc.) with timestamps and personal notes  | `.md`             |
| `Index.md`          | How to read the folder: start here, reading order, what to skip       | `.md`             |

Conventions:
- One file = one subtopic (e.g. `cheatsheets/llama-server-flags.md`).
- Cheatsheets must start with a "TL;DR" block and a "verified on version/date" line.
- `videos.md` entries: `- [Title](url) — why it's good, key timestamps, my notes`.
- Status markers in this file: `[ ]` not started · `[~]` in progress · `[x]` done.

## 2. Folder layout

```
LocalLLM/
├── README.md                     # What this repo is + how to navigate
├── Structure.md                  # THIS FILE — map of the repo
├── slide.md                      # Presentation plan: "Running LLMs locally" talk
├── 00-foundations/               # Core concepts: tokens, params, architecture, quantization
├── 01-huggingface/               # Model hub, searching, downloading, transformers
├── 02-hardware-and-vram/         # VRAM/RAM math, GPUs, CPU inference, buying guide
├── 03-ollama/                    # Install, CLI, Modelfile, env vars, API
├── 04-llama-cpp/                 # Build, all tools, all flags, GGUF workflow
├── 05-vllm/                      # Install, vllm serve, offline inference, args
├── 06-sampling-and-prompting/    # Sampling params, chat templates, perf metrics
├── 07-multimodal-vision/         # Vision models, how they work, running them locally
└── 08-apis-and-clients/          # OpenAI-compatible API, curl/Python, frontends
```

---

## 3. Topic breakdown

### 00 — Foundations `[x]`
Core concepts. Everything else assumes this is understood.
- [x] **What is a token**: BPE/byte-level tokenization, tokens vs words, why it matters for context length and cost; `tiktoken` / HF tokenizer demo
- [x] **What are parameters**: weights, why "7B" means 7 billion, parameter count vs intelligence, dense vs sparse
- [x] **Model architecture basics**: Transformer, attention, layers, hidden size, heads — enough to understand VRAM math
- [x] **MoE (Mixture of Experts)**: what it is, active vs total params, why DeepSeek/Qwen MoE are fast but big
- [x] **MTP (Multi-Token Prediction)**: what it is, which models use it (DeepSeek-V3/R1), impact on inference
- [x] **Context window**: what it is, sliding windows, why it's not free (KV cache)
- [x] **Inference pipeline**: prompt processing (prefill) vs token generation (decode), what "tokens/sec" really means
- [x] **Model formats**: safetensors vs GGUF vs others, what quantization is (FP16/BF16/INT8/INT4, Q4_K_M etc.), quant levels and quality tradeoffs
- [x] **System prompts & chat templates**: what a chat template is, why it matters for quality

### 01 — Hugging Face `[x]`
- [x] **The Hub**: how models are organized (org/model, tags, task, license) — `notes/searching-models.md`
- [x] **Searching models**: filters (task, params, license, quantization, GGUF), comparing model cards, reading benchmarks honestly — `notes/searching-models.md`, `cheatsheets/hf-search.md`
- [x] **Model cards**: architecture, context, license, recommended usage — `notes/searching-models.md`
- [x] **Downloading**: `hf` CLI, `git lfs`, Python `huggingface_hub`, gated models & tokens — `notes/useful-tools.md`, `cheatsheets/hf-download.md`
- [x] **`transformers` library**: `pipeline()` for quick tests, `AutoModelForCausalLM`, when to use it vs a dedicated runtime — `notes/useful-tools.md`
- [x] **GGUF repos on HF**: where to find community quantizations (ggml-org, bartowski, unsloth, mradermacher), which quants to pick — `notes/formats-and-quantization.md`
- [x] **Licenses**: MIT vs Apache-2.0 vs Llama community license vs research-only — what you can and can't do — `notes/searching-models.md`
- [x] **Formats on the Hub**: GGUF vs safetensors vs MLX/ONNX, GPTQ/AWQ/FP8, how to tell what a repo is — `notes/formats-and-quantization.md`, `cheatsheets/gguf-filenames.md`
- [x] **MTP models**: where MTP lives (config flag, embedded tensors, sidecar/assistant files), how to check & run them (llama.cpp `--spec-type draft-mtp`, vLLM) — `notes/mtp-models.md`
- [x] **mmproj (vision)**: what it is, naming, where to find it, how to use it (llama.cpp/Ollama/LM Studio) — `notes/mmproj-vision.md`
- [x] **Inspecting weights**: file listing, `config.json`, GGUF/safetensors tensor inspection, checksums — `notes/inspecting-weights.md`, `cheatsheets/inspect-weights.md`
- [x] **Extra tools**: full `hf` CLI, cache management, aria2c, hf-mirror, HF API for scripting — `notes/useful-tools.md`

### 02 — Hardware & VRAM `[x]`
- [x] **The VRAM formula**: `weights + KV cache + activations/overhead` with worked examples (7B, 13B, 70B at Q4/Q8/FP16) — `notes/vram-formula.md`
- [x] **Quantization math**: bytes per weight, table of quant → size → quality (Q4_K_M ≈ 0.61 bytes/param measured, etc.) — `notes/quantization-math.md`
- [x] **KV cache math**: how context length and batch size eat VRAM; why 128k context on a 7B model can OOM — `notes/kv-cache-math.md`
- [x] **VRAM vs RAM vs CPU**: what happens when the model doesn't fit in VRAM, offloading, expected speeds (GPU vs CPU tokens/sec) — `notes/vram-vs-ram-cpu.md`
- [x] **GPU guide**: consumer cards (RTX 30/40/50 series VRAM table), what model size each card runs comfortably — `notes/gpu-guide.md`
- [x] **Apple Silicon / unified memory**: why Macs run big models, memory bandwidth limits, swapping — `notes/apple-silicon.md`
- [x] **Multi-GPU**: when and how to split a model across cards — `notes/multi-gpu.md`
- [x] **Cheatsheet**: "I have X GB VRAM → which models/quants fit" decision table — `cheatsheets/vram-budget.md`

### 03 — Ollama `[x]`
> All examples in this section are Docker-based (docker CLI / compose) — the official `ollama/ollama` image is the install.
- [x] **Install**: Docker-first (official image, `docker run` + compose), image tags, updating, uninstall; native install (script + systemd) for reference — `notes/installation.md`
- [x] **Core commands**: `run`, `pull`, `ls`, `ps`, `stop`, `show`, `create`, `serve`, `launch`, `help` via `docker exec` (note: no `bench` in the current CLI) — `notes/core-commands.md`, `cheatsheets/ollama-commands.md`
- [x] **Model management**: `create`, `rm`, `cp`, `push`, pull from custom registries (`OLLAMA_REGISTRY`), tags — `notes/model-management.md`
- [x] **Modelfile**: `FROM` (model/GGUF/safetensors), `PARAMETER`, `SYSTEM`, `TEMPLATE`, `MESSAGE`, `REQUIRES`, `CAPABILITY` (`ADAPTER` is legacy) — `notes/modelfile.md`
- [x] **Configuration (env vars)**: full table with defaults (`OLLAMA_HOST`, `OLLAMA_MODELS`, `OLLAMA_NUM_PARALLEL`, `OLLAMA_MAX_LOADED_MODELS`, `OLLAMA_KV_CACHE_TYPE`, `OLLAMA_CONTEXT_LENGTH`, `OLLAMA_FLASH_ATTENTION`, `OLLAMA_NUM_GPU`, `OLLAMA_KEEP_ALIVE`, `OLLAMA_MAX_QUEUE`, …) — `notes/configuration.md`, `cheatsheets/ollama-env-vars.md`
- [x] **systemd**: `systemctl edit ollama` to set env vars persistently (native install) — `notes/configuration.md`
- [x] **REST API**: `/api/generate`, `/api/chat`, `/api/tags`, `/api/ps` with curl, streaming, keep_alive, OpenAI-compatible — `notes/rest-api.md`, `cheatsheets/ollama-api.md`
- [x] **Vision models in Ollama**: running llava / gemma3 / qwen-vl with image input (CLI + base64 API) — `notes/vision-models.md`
- [x] **Docker**: official image, GPU (NVIDIA/AMD/Vulkan), mounting models, exposing the port, compose, multi-instance — `notes/docker.md`, `cheatsheets/ollama-docker.md`

### 04 — llama.cpp `[x]`
> All examples in this section are Docker-based — the official `ghcr.io/ggml-org/llama.cpp` images are the install (CMake native build is reference-only; the Makefile build is gone).
- [x] **Build**: Docker-first (image matrix full/light/server × CPU/CUDA/ROCm/Vulkan/SYCL/MUSA/OpenVINO), GPU prerequisites, local image builds; CMake native build (CPU, CUDA, ROCm, Vulkan, SYCL, MUSA, BLAS) for reference — `notes/installation.md`, `notes/docker.md`, `cheatsheets/llama-cpp-docker.md`
- [x] **Tool inventory**: current `tools/` (server, cli, completion, quantize, imatrix, bench, perplexity, export-lora, gguf-split, …), removed tools (`llama-embeddings`, `llama-save-load-state`), unified `llama` binary (`llama cli`, `llama serve`) — `notes/tools-inventory.md`
- [x] **llama-server — full flag reference** (the big cheatsheet):
  - hosting: `--host`, `--port`, `--api-key`, `--cors`
  - model: `--model`, `-hf`, `--lora`, `--alias`
  - memory: `--n-gpu-layers`, `--n-cpu-moe`, `--main-gpu`, `--tensor-split`, `--load-mode` (mmap/mlock), `--cache-type-k/v`
  - context: `--ctx-size`, `--n-predict`, `--parallel`, `--batch`, `--ubatch`, `--flash-attn`
  - sampling: `--temp`, `--top-k`, `--top-p`, `--min-p`, `--repeat-penalty`, `--seed`, `--stop`
  - advanced: `--jinja`, `--chat-template`, `--reasoning`, `--spec-type` (draft models / speculative decoding), `--metrics`, router/agent modes — `notes/llama-server.md`, `cheatsheets/llama-server-flags.md`
- [x] **llama-cli — interactive use**: key flags, `-p`/`--single-turn`, `--reverse-prompt`, `--verbose-prompt`, reasoning models — `notes/llama-cli.md`, `cheatsheets/llama-cli-flags.md`
- [x] **Quantizing your own models**: `convert_hf_to_gguf.py` → `llama-quantize`, imatrix, quant selection, mmproj — `notes/quantization.md`
- [x] **Benchmarking**: `llama-bench` to compare quants/flags on your hardware — `notes/benchmarking.md`
- [x] **Speculative decoding**: draft models, EAGLE-3/DFlash/DSpark, MTP heads, ngram (no extra model), when it helps — `notes/speculative-decoding.md`
- [x] **LoRA**: loading adapters, runtime switching, merging with `llama-export-lora` — `notes/lora.md`
- [x] **OpenAI-compatible API**: endpoints, chat template behavior, `/v1/models`, `/v1/chat/completions`, Anthropic `/v1/messages`, native endpoints — `notes/api.md`, `cheatsheets/llama-cpp-commands.md`
- [x] **Docker**: image matrix, GPU (NVIDIA/AMD/Vulkan/Intel), mounting models, `-hf` pulls, compose, `LLAMA_ARG_*` env vars — `notes/docker.md`, `cheatsheets/llama-cpp-docker.md`

### 05 — vLLM `[x]`
- [x] **What vLLM is**: PagedAttention, continuous batching, why it's fast for serving — `notes/what-is-vllm.md`
- [x] **Install**: pip (with CUDA), Docker image, conda; hardware requirements — `notes/installation.md`, `notes/docker.md`, `cheatsheets/vllm-docker.md`
- [x] **Quickstart**: `vllm serve <model>` → OpenAI-compatible server in one command — `notes/quickstart.md`
- [x] **`vllm serve` — key flags**: `--tensor-parallel-size`, `--pipeline-parallel-size`, `--gpu-memory-utilization`, `--max-model-len`, `--max-num-seqs`, `--quantization` (awq, gptq, fp8, bitsandbytes…), `--dtype`, `--enable-prefix-caching`, `--speculative-config`, `--served-model-name`, `--port`, `--api-key` — `notes/serve-flags.md`, `cheatsheets/vllm-serve-flags.md`
- [x] **Offline inference**: Python `LLM` class, `SamplingParams`, batched generation — `notes/offline-inference.md`
- [x] **Model support**: which architectures/quants are supported, vision models (Qwen2-VL etc.), audio — `notes/model-support.md`
- [x] **Multi-GPU**: tensor parallelism in practice, examples (70B on 2×24GB) — `notes/multi-gpu.md`
- [x] **Benchmarking**: `vllm bench` (throughput, latency, TTFT) — `notes/benchmarking.md`, `cheatsheets/vllm-commands.md`
- [x] **Metrics & monitoring**: Prometheus `/metrics`, what to watch — `notes/metrics.md`
- [x] **vLLM vs llama.cpp vs Ollama**: when to use which (cheatsheet) — `cheatsheets/vllm-vs-llamacpp-vs-ollama.md`

### 06 — Sampling & Prompting `[x]`
- [x] **Sampling parameters**: `temperature`, `top_p`, `top_k`, `min_p`, `typical_p`, `repetition_penalty`, `frequency/presence_penalty`, `seed` — what each does + recommended ranges — `notes/sampling-parameters.md`, `cheatsheets/sampling-params.md`
- [x] **Reasoning models**: thinking budgets, `reasoning_effort`, how to parse thinking blocks (llama.cpp `--reasoning*`, Ollama `think`, vLLM `--reasoning-parser`) — `notes/reasoning-models.md`, `cheatsheets/reasoning-controls.md`
- [x] **Chat templates**: how the prompt is assembled, `--jinja` in llama.cpp, `SYSTEM` in Ollama, what breaks when the template is wrong — `notes/chat-templates.md`
- [x] **Performance metrics**: TTFT (time to first token), tokens/sec decode, prompt processing speed, how to measure them — `notes/performance-metrics.md`, `cheatsheets/perf-metrics.md`
- [x] **Prompting basics for local models**: system prompt design, few-shot, structured output, why local models need clearer instructions — `notes/prompting-basics.md`

### 07 — Multimodal & Vision `[x]`
- [x] **How vision LLMs work**: image → patches → visual tokens, how many "tokens" an image costs, impact on context — `notes/how-vision-llms-work.md`, `cheatsheets/image-token-cost.md`
- [x] **Model landscape**: LLaVA, Qwen2.5-VL / Qwen3-VL, Gemma 3, Llama 3.2 Vision, MiniCPM-V — sizes and capabilities — `notes/vision-models.md`, `cheatsheets/vision-models.md`
- [x] **Running vision models**: in Ollama (image flag), llama.cpp (`--image` + mmproj), vLLM (multimodal input) — with working examples — `notes/running-vision-models.md`
- [x] **VRAM for vision models**: why they need more headroom (encoder weights, visual KV, activations) — `notes/vision-vram.md`
- [x] **Audio models (bonus)**: Whisper / faster-whisper / whisper.cpp, Qwen2-Audio, local speech pipeline — `notes/audio-models.md`
- [x] **Embedding models (bonus)**: what they are, nomic-embed, bge, Qwen3-Embedding, RAG in 5 steps — `notes/embedding-models.md`, `cheatsheets/audio-embeddings.md`

### 08 — APIs & Clients `[x]`
- [x] **OpenAI-compatible API**: why it's the de-facto standard, endpoints, per-runtime support matrix + gotchas — `notes/openai-compatible-api.md`, `cheatsheets/openai-api.md`
- [x] **curl examples**: health, chat, streaming (SSE), JSON mode, embeddings, tools, images — `notes/curl-cookbook.md`, `cheatsheets/curl-cookbook.md`
- [x] **Python**: `openai` client pointed at local base_url, streaming, async, multi-runtime config — `notes/python-clients.md`
- [x] **Frontends**: Open WebUI, AnythingLLM, LibreChat, SillyTavern — install + point at Ollama/llama.cpp/vLLM — `notes/frontends.md`, `cheatsheets/frontends.md`
- [x] **MCP servers (bonus)**: what MCP is, running local MCP servers with your local LLM, writing your own — `notes/mcp-servers.md`

---

## 4. Roadmap (suggested order)

1. **00 Foundations** → **02 Hardware & VRAM** (you need the math before picking models)
2. **01 Hugging Face** (finding and downloading models)
3. **03 Ollama** (easiest runtime — get something running fast)
4. **04 llama.cpp** (deepest control — GGUF workflow)
5. **05 vLLM** (serving performance)
6. **06 Sampling** → **07 Vision**
7. **08 APIs & clients**

## 5. Naming conventions

- Folders: `NN-topic-name` (kebab-case, 2-digit prefix = order).
- Files: kebab-case, one subtopic per file, e.g. `cheatsheets/ollama-env-vars.md`.
- Cheatsheets: `TL;DR` → tables → examples → "verified on <version>, <date>".
- Notes: start with a one-paragraph summary, then details, then "my experiments".
- Slides: `<topic>-deck-v<N>.pptx` (or `.md` if markdown-based slides).
- `slide.md` (root): presentation plan for the "Running LLMs locally" talk — 10 slides, each with on-slide content, what to say, and the live demo.
- `Index.md`: one per folder — "start here" + numbered reading order + quick reference; update it when files are added or removed.
