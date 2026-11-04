# Cheatsheet — vLLM vs llama.cpp vs Ollama

**TL;DR.** Same models, three engines, three jobs: **Ollama** = zero-friction local use (one command, model registry); **llama.cpp** = single-user max control (GGUF quants, CPU/hybrid, per-flag tuning, MTP drafts); **vLLM** = high-throughput serving (many concurrent requests, OpenAI API, multi-GPU, HF weights). If you have one user and one machine, llama.cpp/Ollama win on simplicity and per-token speed. If you have many users (or want an OpenAI-compatible endpoint + Prometheus), vLLM wins on throughput. All three run in Docker in this repo. Verified 2026-11-03.

## One-line positioning

| Tool | You are… |
|---|---|
| Ollama (03) | "I want a model running in 30 seconds" |
| llama.cpp (04) | "I want the best single-user performance I can tune" |
| vLLM (05) | "I want to serve N concurrent users / an OpenAI API / multiple GPUs" |

## Feature matrix

| | Ollama | llama.cpp | vLLM |
|---|---|---|---|
| Weights | GGUF (registry + local) | GGUF (local, `-hf` pull) | **HF safetensors** (registry, local) + GGUF |
| Quant control | Modelfile-level (basic) | per-layer, imatrix, IQ quants | repo-level (AWQ/GPTQ/FP8/GGUF/bnb) |
| Concurrency model | 1 request at a time (queue) | `--parallel` slots, static-ish | **continuous batching** (per-iteration) |
| OpenAI API | yes (`/v1`) | yes (`/v1`) | yes (`/v1`, broadest surface) |
| Anthropic API | no | yes (`/v1/messages`) | no |
| Multi-GPU | basic | `--n-gpu-layers` split, tensor via `--tensor-split` | **tensor/pipeline/data/expert parallel** |
| CPU / hybrid | yes | **best-in-class** (ARM, Vulkan, SYCL, MUSA) | weak (pip, testing only) |
| Speculative decoding | no (as of 04 notes) | draft/EAGLE-3/MTP/ngram | ngram/suffix/EAGLE/MTP/draft/DFlash |
| Prefix caching | no | no | **yes (default, hash-based)** |
| LoRA | no | yes (load/switch/merge) | yes (multi-LoRA serving) |
| Vision/audio | yes (models) | yes (mmproj) | yes (VLMs, Whisper, Qwen2-Audio) |
| Metrics | basic | Prometheus `/metrics` | **Prometheus-first + Grafana dashboard** |
| Batched offline | no | `llama-bench`, completion | `LLM` class, `run-batch`, `bench throughput` |
| Benchmarking | no | `llama-bench` (pp/tg) | `vllm bench serve/throughput/latency` |
| Ecosystem | `ollama pull`, registry, Modelfile | `convert_hf_to_gguf`, `llama-quantize`, imatrix | HF integration, recipes, speculative ecosystem |
| Docker image | `ollama/ollama` | `ghcr.io/ggml-org/llama.cpp` | `vllm/vllm-openai` |

## When to use which

**Ollama**
- Trying models fast; `ollama run qwen3:8b` and go.
- Small team, no API surface needed, Open WebUI-style frontend.
- You want a model *registry* experience (pull by tag, no file management).

**llama.cpp**
- **Single user, one machine** — per-token decode speed is competitive or best, especially on consumer GPUs + CPU offload.
- You live in **GGUF** (your 04 workflow): quantize with imatrix, pick Q4_K_M/Q5_K_M, tune `--n-gpu-layers`, `--flash-attn`, `--cache-type-k/v`.
- CPU-first or weird hardware (Vulkan, SYCL, MUSA, ARM).
- MTP draft models on Qwen3-class models (your `testcommands` setup).
- Anthropic-compatible endpoint needed.

**vLLM**
- **More than one concurrent user** — continuous batching is where it pulls ahead; at 1 request it's often *slower* per token than llama.cpp, at 16+ it's much faster in aggregate.
- You need the **OpenAI API** as a stable contract (LangChain, Open WebUI, agents) with the broadest feature surface (tools, reasoning parsers, multimodal).
- **Multi-GPU** serving (TP on one node, PP across nodes, DP replicas).
- **HF weights** workflow: pull AWQ/FP8/GPTQ checkpoints straight from HF; or reuse your GGUFs (`-q gguf`).
- **Prefix caching** matters (shared system prompts, RAG, agents) — big TTFT wins, on by default.
- **Observability**: Prometheus metrics + Grafana, SLO tracking.
- Batch jobs: offline `LLM` / `run-batch` at engine speed.

## Same model, three ways (the mental experiment)

Take `Qwen3.8-27B` (your current model):

```bash
# 03 — Ollama: registry pull, GGUF under the hood
ollama run qwen3:27b

# 04 — llama.cpp: your quant, your flags, MTP draft
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /m/Qwen3.8-27B-Q4_K_M.gguf --n-gpu-layers -1 \
  --spec-type mtp --spec-draft-model /m/Qwen3.8-27B-MTP.gguf \
  --host 0.0.0.0 --port 8080

# 05 — vLLM: HF weights (or your GGUF), OpenAI API, batching
docker run --rm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen3.8-27B-Instruct-FP8 --max-model-len 32k
# or with your GGUF: --model /m/Qwen3.8-27B-Q4_K_M.gguf -q gguf
```

Then benchmark honestly: `llama-bench` (04) for per-stream speed, `vllm bench serve` (05) with `--max-concurrency 1,4,16,32` for the concurrency curve. The crossover point (where vLLM's aggregate throughput overtakes llama.cpp) depends on your GPU — measure it, don't assume.

## Hybrid setups (real world)

- **llama.cpp for interactive, vLLM for batch/API** — same GGUF weights, two containers, different jobs.
- **Ollama for exploration, vLLM for the production endpoint** — try a model in 30 seconds, then serve the HF version.
- **vLLM TP=2 for a model that doesn't fit on one card** — llama.cpp can split layers too (`--n-gpu-layers`), but vLLM's TP is cleaner for multi-node.

## Decision tree

```
How many concurrent users?
├── 1 (you)
│   ├── want it now, no tuning? → Ollama
│   └── want max control/speed, GGUF, CPU/hybrid, MTP? → llama.cpp
└── >1 (or "API endpoint" is the point)
    ├── multi-GPU / multi-node? → vLLM (TP/PP/DP)
    ├── need OpenAI API + metrics + prefix caching? → vLLM
    └── modest concurrency, already in GGUF? → llama.cpp --parallel N (still fine)
Weird hardware (Vulkan/SYCL/MUSA/ARM)? → llama.cpp
Batch jobs / evals / data gen? → vLLM offline (LLM, run-batch)
```

## Gotchas

- **"Faster" is workload-shaped** — vLLM wins on throughput under concurrency; llama.cpp often wins single-stream decode. Benchmark *your* workload (04 + 05 both have the tools).
- **Weight format is the hidden cost** — vLLM prefers HF safetensors (AWQ/FP8); llama.cpp prefers GGUF. Your GGUFs work in both; your AWQ checkpoints work in vLLM only.
- **Context limits differ** — same model, different max-context defaults; set `--ctx-size` / `--max-model-len` explicitly when comparing.
- **Don't mix metrics across tools** — llama.cpp `tg` t/s (single stream) vs vLLM output tok/s (aggregate) answer different questions.
- **One model per vLLM instance** — multi-model needs replicas/ports; Ollama/llama.cpp servers also serve one model (llama.cpp: restart or new port).
