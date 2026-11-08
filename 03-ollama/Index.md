# Index — 03-ollama

**How to read this folder.** Ollama is the fastest way to get a model running — read in the order you'd actually set it up: install → use → manage → configure → extend → talk to it programmatically. The Docker deep dive comes *after* you're comfortable with the CLI (it's the same ideas, containerized).

## Start here

[notes/installation.md](notes/installation.md) — Docker-first install, the path this repo assumes everywhere.

## Reading order

1. [notes/installation.md](notes/installation.md) — get Ollama running (Docker-first)
2. [notes/core-commands.md](notes/core-commands.md) — `list`, `run`, `ps`, `stop` — the daily loop
3. [notes/model-management.md](notes/model-management.md) — pull/create/tags/registries, managing what's on disk
4. [notes/configuration.md](notes/configuration.md) — env vars (GPU, context, keep-alive) + systemd
5. [notes/modelfile.md](notes/modelfile.md) — building custom models (system prompt, params baked in)
6. [notes/rest-api.md](notes/rest-api.md) — the native API (`/api/chat`, `/api/generate`) — programmatic use
7. [notes/docker.md](notes/docker.md) — Docker deep dive: ports, volumes, GPU passthrough, LAN *(after you're comfortable)*
8. [notes/vision-models.md](notes/vision-models.md) — running vision models in Ollama *(bridge into [07-multimodal-vision](../07-multimodal-vision/))*

## Quick reference (cheatsheets)

- [cheatsheets/ollama-commands.md](cheatsheets/ollama-commands.md) — every command with examples
- [cheatsheets/ollama-env-vars.md](cheatsheets/ollama-env-vars.md) — the env vars that matter
- [cheatsheets/ollama-api.md](cheatsheets/ollama-api.md) — native API endpoints + curl examples
- [cheatsheets/ollama-docker.md](cheatsheets/ollama-docker.md) — docker run variants (GPU, ports, volumes)

## Sources & videos

- [docs/references.md](docs/references.md) — official docs
- [videos.md](videos.md) — curated videos
