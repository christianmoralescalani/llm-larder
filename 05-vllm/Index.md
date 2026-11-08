# Index — 05-vllm

**How to read this folder.** vLLM is the serving-performance runtime — read in the order "why → run it → tune it → scale it → measure it". If you're still deciding whether vLLM is for you, read the comparison cheatsheet *first*, then come back.

## Start here

[cheatsheets/vllm-vs-llamacpp-vs-ollama.md](cheatsheets/vllm-vs-llamacpp-vs-ollama.md) — 2 minutes: when vLLM wins and when it doesn't. Then [notes/what-is-vllm.md](notes/what-is-vllm.md).

## Reading order

1. [notes/what-is-vllm.md](notes/what-is-vllm.md) — what it is, PagedAttention, when it beats llama.cpp
2. [notes/quickstart.md](notes/quickstart.md) — `vllm serve` in one command
3. [notes/installation.md](notes/installation.md) — Docker-first install, images, GPU
4. [notes/serve-flags.md](notes/serve-flags.md) — the flags that matter (tensor parallel, GPU memory util, max model len)
5. [notes/docker.md](notes/docker.md) — Docker deep dive: volumes, GPU, multi-container
6. [notes/model-support.md](notes/model-support.md) — what it can load (HF ids, GGUF, vision, MoE)
7. [notes/offline-inference.md](notes/offline-inference.md) — the Python `LLM` class, no server
8. [notes/multi-gpu.md](notes/multi-gpu.md) — tensor parallelism in practice
9. [notes/metrics.md](notes/metrics.md) — `/metrics`, monitoring, what to watch
10. [notes/benchmarking.md](notes/benchmarking.md) — `vllm bench`: TTFT, throughput, good numbers

## Quick reference (cheatsheets)

- [cheatsheets/vllm-commands.md](cheatsheets/vllm-commands.md) — every command with examples
- [cheatsheets/vllm-serve-flags.md](cheatsheets/vllm-serve-flags.md) — the flags, grouped
- [cheatsheets/vllm-docker.md](cheatsheets/vllm-docker.md) — docker run variants
- [cheatsheets/vllm-vs-llamacpp-vs-ollama.md](cheatsheets/vllm-vs-llamacpp-vs-ollama.md) — the decision table

## Sources & videos

- [docs/references.md](docs/references.md) — official docs
- [videos.md](videos.md) — curated videos
