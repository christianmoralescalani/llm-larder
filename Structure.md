# LocalLLM — Repository Structure

> Knowledge base for running LLMs locally: notes, videos, slides (diapos), cheatsheets, and docs.
> This file is the **single source of truth** for how the repository is organized. Update it whenever a topic is added, split, or renamed.

---

## 1. Content types

Every topic folder uses the same 5 content types, so navigation is always predictable:

| Folder / file       | What goes there                                                        | Format            |
|---------------------|------------------------------------------------------------------------|-------------------|
| `notes/`            | Long-form written notes, explanations, step-by-step guides            | `.md`             |
| `cheatsheets/`      | One-page references: commands, flags, formulas, tables                | `.md`             |
| `slides/`           | Slide decks (diapos) for presentations                                | `.pptx` / `.pdf` / `.md` |
| `docs/`             | External docs, official references, saved PDFs, API specs             | links / `.pdf`    |
| `videos.md`         | Curated video list (YouTube etc.) with timestamps and personal notes  | `.md`             |

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
├── 00-foundations/               # Core concepts: tokens, params, architecture, quantization
├── 01-huggingface/               # Model hub, searching, downloading, transformers
├── 02-hardware-and-vram/         # VRAM/RAM math, GPUs, CPU inference, buying guide
├── 03-ollama/                    # Install, CLI, Modelfile, env vars, API
├── 04-llama-cpp/                 # Build, all tools, all flags, GGUF workflow
├── 05-vllm/                      # Install, vllm serve, offline inference, args
├── 06-sampling-and-prompting/    # Sampling params, chat templates, perf metrics
├── 07-multimodal-vision/         # Vision models, how they work, running them locally
├── 08-model-families/            # Llama, Qwen, Mistral, Gemma, DeepSeek… how to choose
├── 09-apis-and-clients/          # OpenAI-compatible API, curl/Python, frontends
├── 10-other-runtimes/            # LM Studio, text-gen-webui, LocalAI, SGLang, etc.
├── 11-deployment/                # Docker, LAN exposure, systemd, monitoring, security
├── 12-troubleshooting/           # Common errors, OOM, CUDA issues, FAQ
└── 13-glossary-and-links/        # Glossary + curated external resources
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

### 01 — Hugging Face `[ ]`
- [ ] **The Hub**: how models are organized (org/model, tags, task, license)
- [ ] **Searching models**: filters (task, params, license, quantization, GGUF), comparing model cards, reading benchmarks honestly
- [ ] **Model cards**: architecture, context, license, recommended usage
- [ ] **Downloading**: `huggingface-cli` (`hf` CLI), `git lfs`, Python `huggingface_hub`, gated models & tokens
- [ ] **`transformers` library**: `pipeline()` for quick tests, `AutoModelForCausalLM`, when to use it vs a dedicated runtime
- [ ] **GGUF repos on HF**: where to find community quantizations (bartowski, unsloth, mradermacher), which quants to pick
- [ ] **Licenses**: MIT vs Apache-2.0 vs Llama community license vs research-only — what you can and can't do

### 02 — Hardware & VRAM `[ ]`
- [ ] **The VRAM formula**: `weights + KV cache + activations/overhead` with worked examples (7B, 13B, 70B at Q4/Q8/FP16)
- [ ] **Quantization math**: bytes per weight, table of quant → size → quality (Q4_K_M ≈ 0.57 bytes/param, etc.)
- [ ] **KV cache math**: how context length and batch size eat VRAM; why 128k context on a 7B model can OOM
- [ ] **VRAM vs RAM vs CPU**: what happens when the model doesn't fit in VRAM, offloading, expected speeds (GPU vs CPU tokens/sec)
- [ ] **GPU guide**: consumer cards (RTX 30/40/50 series VRAM table), what model size each card runs comfortably
- [ ] **Apple Silicon / unified memory**: why Macs run big models, memory bandwidth limits, swapping
- [ ] **Multi-GPU**: when and how to split a model across cards
- [ ] **Cheatsheet**: "I have X GB VRAM → which models/quants fit" decision table

### 03 — Ollama `[ ]`
- [ ] **Install**: Linux (script + systemd), Windows, macOS; updating; uninstall
- [ ] **Core commands**: `ollama run`, `pull`, `list`, `ps`, `stop`, `show`, `bench`, `serve`, `help`
- [ ] **Model management**: `create`, `delete`, `push`, `pull` from custom registries, tags
- [ ] **Modelfile**: `FROM`, `PARAMETER`, `SYSTEM`, `TEMPLATE`, `ADAPTER` — building custom models
- [ ] **Configuration (env vars)**: `OLLAMA_HOST`, `OLLAMA_MODELS`, `OLLAMA_NUM_PARALLEL`, `OLLAMA_MAX_LOADED_MODELS`, `OLLAMA_KV_CACHE`, `OLLAMA_CONTEXT_LENGTH`, `OLLAMA_FLASH_ATTENTION`, `OLLAMA_NUM_GPU`, `OLLAMA_KEEP_ALIVE`, `OLLAMA_MAX_QUEUE`… full table with defaults
- [ ] **systemd**: `systemctl edit ollama` to set env vars persistently
- [ ] **REST API**: `/api/generate`, `/api/chat`, `/api/tags`, `/api/ps` with curl examples
- [ ] **Vision models in Ollama**: running llava / gemma3 / qwen-vl with image input
- [ ] **Docker**: official image, mounting models, exposing the port

### 04 — llama.cpp `[ ]`
- [ ] **Build**: CMake (CPU, CUDA, ROCm, Vulkan, SYCL), prebuilt binaries, `make llamacpp`
- [ ] **Tool inventory**: `llama-cli`, `llama-server`, `llama-quantize`, `llama-bench`, `llama-perplexity`, `llama-embeddings`, `llama-save-load-state`, `llama-export-lora`, `convert_hf_to_gguf.py`
- [ ] **llama-server — full flag reference** (the big cheatsheet):
  - hosting: `--host`, `--port`, `--api-key`, `--cors`
  - model: `--model`, `--lora`, `--gguf-model`, `--huggingface-repo`, `--huggingface-file`
  - memory: `--n-gpu-layers`, `--n-cpu-moe`, `--main-gpu`, `--tensor-split`, `--mlock`, `--mmap`, `--no-mmap`, `--cache-type-k/v`
  - context: `--ctx-size`, `--n-predict`, `--parallel`, `--batch`, `--ubatch`, `--flash-attn`
  - sampling: `--temp`, `--top-k`, `--top-p`, `--min-p`, `--repeat-penalty`, `--seed`, `--stop`
  - advanced: `--jinja`, `--chat-template`, `--spec-type` (draft models / speculative decoding), `--rpc`, `--port` metrics
- [ ] **llama-cli — interactive use**: key flags, `--interactive-first`, `--prompt`, reasoning models
- [ ] **Quantizing your own models**: `convert_hf_to_gguf.py` → `llama-quantize`, choosing imatrix, quant selection
- [ ] **Benchmarking**: `llama-bench` to compare quants/flags on your hardware
- [ ] **Speculative decoding**: draft models, self-speculative, when it helps
- [ ] **LoRA**: loading adapters, combining with base models
- [ ] **OpenAI-compatible API**: endpoints, chat template behavior, `/v1/models`, `/v1/chat/completions`

### 05 — vLLM `[ ]`
- [ ] **What vLLM is**: PagedAttention, continuous batching, why it's fast for serving
- [ ] **Install**: pip (with CUDA), Docker image, conda; hardware requirements
- [ ] **Quickstart**: `vllm serve <model>` → OpenAI-compatible server in one command
- [ ] **`vllm serve` — key flags**: `--tensor-parallel-size`, `--pipeline-parallel-size`, `--gpu-memory-utilization`, `--max-model-len`, `--max-num-seqs`, `--quantization` (awq, gptq, fp8, bitsandbytes…), `--dtype`, `--enable-prefix-caching`, `--speculative-config`, `--served-model-name`, `--port`, `--api-key`
- [ ] **Offline inference**: Python `LLM` class, `SamplingParams`, batched generation
- [ ] **Model support**: which architectures/quants are supported, vision models (Qwen2-VL etc.), audio
- [ ] **Multi-GPU**: tensor parallelism in practice, examples (70B on 2×24GB)
- [ ] **Benchmarking**: `vllm bench` (throughput, latency, TTFT)
- [ ] **Metrics & monitoring**: Prometheus `/metrics`, what to watch
- [ ] **vLLM vs llama.cpp vs Ollama**: when to use which (cheatsheet)

### 06 — Sampling & Prompting `[ ]`
- [ ] **Sampling parameters**: `temperature`, `top_p`, `top_k`, `min_p`, `typical_p`, `repetition_penalty`, `frequency/presence_penalty`, `seed` — what each does + recommended ranges
- [ ] **Reasoning models**: thinking budgets, `reasoning_effort`, how to parse thinking blocks
- [ ] **Chat templates**: how the prompt is assembled, `--jinja` in llama.cpp, `SYSTEM` in Ollama, what breaks when the template is wrong
- [ ] **Performance metrics**: TTFT (time to first token), tokens/sec decode, prompt processing speed, how to measure them
- [ ] **Prompting basics for local models**: system prompt design, few-shot, why local models need clearer instructions

### 07 — Multimodal & Vision `[ ]`
- [ ] **How vision LLMs work**: image → patches → visual tokens, how many "tokens" an image costs, impact on context
- [ ] **Model landscape**: LLaVA, Qwen2.5-VL / Qwen3-VL, Gemma 3, Llama 3.2 Vision, MiniCPM-V — sizes and capabilities
- [ ] **Running vision models**: in Ollama (image flag), llama.cpp (`--image`), vLLM (multimodal input) — with working examples
- [ ] **VRAM for vision models**: why they need more headroom
- [ ] **Audio models (bonus)**: Whisper / Qwen2-Audio basics, running locally
- [ ] **Embedding models (bonus)**: what they are, `nomic-embed`, `bge`, use for RAG

### 08 — Model Families `[ ]`
- [ ] **Llama 3 / 3.1 / 3.2 / 3.3 (Meta)**: sizes, context, licenses
- [ ] **Qwen 2.5 / 3 (Alibaba)**: dense + MoE, tool use, why popular
- [ ] **Mistral / Mixtral (Mistral AI)**: sizes, Apache license
- [ ] **Gemma 3 (Google)**: sizes, vision, license
- [ ] **DeepSeek V3 / R1**: MoE + MTP, reasoning, running them (VRAM reality check)
- [ ] **Small models**: Phi, SmolLM, Gemma small — for laptops/RAG
- [ ] **Choosing a model**: size vs task matrix (coding, chat, RAG, agents), license constraints
- [ ] **Cheatsheet**: "best local model per VRAM tier" table (8 / 12 / 16 / 24 / 48 GB)

### 09 — APIs & Clients `[ ]`
- [ ] **OpenAI-compatible API**: why it's the de-facto standard, endpoints (`/v1/chat/completions`, `/v1/completions`, `/v1/models`)
- [ ] **curl examples**: chat, streaming (SSE), JSON mode
- [ ] **Python**: `openai` client pointed at local base_url, streaming, async
- [ ] **Frontends**: Open WebUI, AnythingLLM, LibreChat, SillyTavern — install + point at Ollama/llama.cpp/vLLM
- [ ] **MCP servers (bonus)**: what MCP is, running local MCP servers with your local LLM

### 10 — Other Runtimes `[ ]`
- [ ] **LM Studio**: GUI, model search/download, server mode (quick start for non-terminal users)
- [ ] **text-generation-webui**: features, model management, extensions
- [ ] **LocalAI**: OpenAI drop-in replacement, multi-model
- [ ] **SGLang**: what it is, when it beats vLLM
- [ ] **TensorRT-LLM**: NVIDIA-only, max performance, when it's worth it
- [ ] **Hugging Face `transformers`**: quick tests, prototyping, when it's enough
- [ ] **Comparison table**: runtime × (ease, speed, features, hardware support)

### 11 — Deployment `[ ]`
- [ ] **Docker**: Ollama / llama.cpp / vLLM images, docker-compose examples, volume for models
- [ ] **systemd services**: running llama-server / vLLM as services with restart
- [ ] **Exposing on LAN**: `OLLAMA_HOST=0.0.0.0`, reverse proxy, why you should not expose to the internet
- [ ] **Security**: API keys, CORS, firewall rules, auth in front of the server
- [ ] **Monitoring**: `nvidia-smi`, `nvtop`, vLLM `/metrics`, Ollama `ps`, log rotation
- [ ] **Auto-loading models**: keep-alive, model switching, warmup

### 12 — Troubleshooting `[ ]`
- [ ] **Out of memory (CUDA OOM)**: diagnosis, fixes (quant, less context, less GPU layers, smaller model)
- [ ] **Slow inference**: is it prefill or decode? RAM vs VRAM, `--n-gpu-layers` mistakes, CPU offload
- [ ] **CUDA / driver errors**: version mismatches, `nvidia-smi` checks, container issues
- [ ] **Model quality issues**: wrong chat template, bad quant, truncated context, thinking models cut off
- [ ] **Model not found / download errors**: HF tokens, gated repos, registry issues
- [ ] **Port conflicts, CORS, connection refused**: quick checks
- [ ] **FAQ**: accumulated Q&A from real sessions
- [ ] **Debugging workflow**: what to check first (checklist)

### 13 — Glossary & Links `[ ]`
- [ ] **Glossary**: token, parameter, quant, KV cache, prefill, decode, TTFT, MoE, MTP, GGUF, LoRA, speculative decoding, context window, embedding, chat template, tensor parallelism…
- [ ] **Curated links**: official docs (HF, Ollama, llama.cpp, vLLM), best tutorials, active community resources
- [ ] **Video playlists**: master list of all `videos.md` entries across the repo

---

## 4. Roadmap (suggested order)

1. **00 Foundations** → **02 Hardware & VRAM** (you need the math before picking models)
2. **01 Hugging Face** (finding and downloading models)
3. **03 Ollama** (easiest runtime — get something running fast)
4. **04 llama.cpp** (deepest control — GGUF workflow)
5. **05 vLLM** (serving performance)
6. **06 Sampling** → **07 Vision** → **08 Model families**
7. **09 APIs & clients** → **10 Other runtimes**
8. **11 Deployment** → **12 Troubleshooting** (fills in as problems appear)
9. **13 Glossary & links** (continuous)

## 5. Naming conventions

- Folders: `NN-topic-name` (kebab-case, 2-digit prefix = order).
- Files: kebab-case, one subtopic per file, e.g. `cheatsheets/ollama-env-vars.md`.
- Cheatsheets: `TL;DR` → tables → examples → "verified on <version>, <date>".
- Notes: start with a one-paragraph summary, then details, then "my experiments".
- Slides: `<topic>-deck-v<N>.pptx` (or `.md` if markdown-based slides).
