# Cheatsheet — Ollama environment variables

**TL;DR.** Ollama is configured with env vars read at server start; in Docker that's compose `environment:`/`env_file` + `docker compose up -d --force-recreate`. The big five: `OLLAMA_HOST` (bind), `OLLAMA_MODELS` (storage), `OLLAMA_CONTEXT_LENGTH` (default 4096), `OLLAMA_KEEP_ALIVE` (default 5m), `OLLAMA_KV_CACHE_TYPE` (f16/q8_0/q4_0). Authoritative list: `docker exec ollama ollama serve --help`. Verified 2026-11-03.

## Full table

### Server / network

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_HOST` | `127.0.0.1:11434` (image: `0.0.0.0:11434`) | listen address |
| `OLLAMA_ORIGINS` | `127.0.0.1, 0.0.0.0, localhost, ::1` | CORS origins (comma-separated) |
| `OLLAMA_MAX_BODY_SIZE` | `100M` | max request body |
| `OLLAMA_REGISTRY` | ollama.com registry | default registry for pull/push |
| `OLLAMA_NO_CLOUD` | `0` | `1` = local-only (no cloud features) |

### Models / memory

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_MODELS` | Docker: `/root/.ollama/models` | model store |
| `OLLAMA_CONTEXT_LENGTH` | `4096` | default context (tokens) |
| `OLLAMA_KEEP_ALIVE` | `5m` | loaded-model TTL (`-1` forever, `0` unload) |
| `OLLAMA_MAX_LOADED_MODELS` | `3×GPUs` (3 CPU) | concurrent models |
| `OLLAMA_NUM_PARALLEL` | `1` | parallel requests per model |
| `OLLAMA_MAX_QUEUE` | `512` | queue before 503 |
| `OLLAMA_NUM_GPU` | `1` | `0` CPU only · `1` first GPU · `-1` all layers to GPU |
| `OLLAMA_FLASH_ATTENTION` | auto | `1` force · `0` off |
| `OLLAMA_KV_CACHE_TYPE` | `f16` | KV cache quant (`q8_0` ≈ ½, `q4_0` ≈ ¼); old name `OLLAMA_KV_CACHE` |
| `OLLAMA_GPU_OVERHEAD` | `0` | VRAM reserved per GPU (bytes) |
| `OLLAMA_VULKAN` | `1` (when GPU visible) | `0` disables Vulkan |
| `GGML_VK_VISIBLE_DEVICES` | all | pick Vulkan devices |
| `JETSON_JETPACK` | — | `5`/`6` (Jetson) |

### Scheduling

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_SCHED_SPREAD` | `0` | `1` = don't spread models across GPUs |
| `OLLAMA_SCHED_BUSY` | `0` | `1` = don't load while busy |
| `OLLAMA_LLM_LIBRARY` | auto | force backend (e.g. `cuda_v12`) |

### Housekeeping

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_NOHISTORY` | `0` | `1` = no chat history |
| `OLLAMA_NOPRUNE` | `0` | `1` = keep unused blobs |
| `OLLAMA_DEBUG` | `0` | `1` = debug logging |
| `OLLAMA_DEBUG_LOG_REQUEST` | `0` | `1` = log request bodies |
| `OLLAMA_LOG_LEVEL` | `info` | debug/info/warn/error/panic |
| `OLLAMA_LOG_FILE` | stderr | log file |
| `HTTPS_PROXY` | — | outbound proxy (never set `HTTP_PROXY`) |

## Compose snippet

```yaml
services:
  ollama:
    image: ollama/ollama:0.40.14
    environment:
      OLLAMA_HOST: "0.0.0.0:11434"
      OLLAMA_CONTEXT_LENGTH: "8192"
      OLLAMA_KEEP_ALIVE: "10m"
      OLLAMA_KV_CACHE_TYPE: "q8_0"
      OLLAMA_FLASH_ATTENTION: "1"
      OLLAMA_NUM_PARALLEL: "1"
      OLLAMA_MAX_LOADED_MODELS: "1"
```

Apply: `docker compose up -d --force-recreate`. Verify: `docker exec ollama env | grep OLLAMA`.

## Verified on

- Ollama docs (FAQ, Docker, CLI) + Docker Hub `ollama/ollama:0.40.14`, 2026-11-03.
