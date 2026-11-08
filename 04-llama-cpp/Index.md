# Index — 04-llama-cpp

**How to read this folder.** llama.cpp is the deepest-control runtime — the most flags in the repo. Read in this order: install → know what each binary is → use the two main ones (CLI, then server) → the workflows (quantize, Docker, benchmark) → the API → advanced speedups. The flag cheatsheets are your daily reference, not first reads.

## Start here

[notes/installation.md](notes/installation.md) — Docker-first (the `server-cuda` image is the path this repo uses).

## Reading order

1. [notes/installation.md](notes/installation.md) — get `llama-server` running on a GGUF
2. [notes/tools-inventory.md](notes/tools-inventory.md) — what every binary does (`llama-cli`, `llama-server`, `llama-quantize`, `llama-bench`, …)
3. [notes/llama-cli.md](notes/llama-cli.md) — interactive use, the flags you'll actually touch
4. [notes/llama-server.md](notes/llama-server.md) — deep dive: ports, `--alias`, `--mmproj`, context, GPU layers
5. [notes/quantization.md](notes/quantization.md) — making your own GGUF quants from safetensors
6. [notes/docker.md](notes/docker.md) — Docker deep dive: images, GPU, volumes, flash-attn
7. [notes/benchmarking.md](notes/benchmarking.md) — `llama-bench`: measuring prompt processing & tok/s
8. [notes/api.md](notes/api.md) — the OpenAI/Anthropic/native endpoints *(bridge into [08-apis-and-clients](../08-apis-and-clients/))*
9. [notes/speculative-decoding.md](notes/speculative-decoding.md) — draft models + MTP for free speed *(advanced)*
10. [notes/lora.md](notes/lora.md) — loading LoRA adapters *(only if you use them)*

## Quick reference (cheatsheets)

- [cheatsheets/llama-cpp-commands.md](cheatsheets/llama-cpp-commands.md) — every command with examples
- [cheatsheets/llama-cli-flags.md](cheatsheets/llama-cli-flags.md) — the flags that matter, grouped
- [cheatsheets/llama-server-flags.md](cheatsheets/llama-server-flags.md) — server flags: model, GPU, context, API
- [cheatsheets/llama-cpp-docker.md](cheatsheets/llama-cpp-docker.md) — docker run variants (the Qwen3.8-27B setup from `testcommands` is here)

## Sources & videos

- [docs/references.md](docs/references.md) — official docs & READMEs
- [videos.md](videos.md) — curated videos
