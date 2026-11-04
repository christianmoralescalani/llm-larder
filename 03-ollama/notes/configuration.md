# Configuration: environment variables (+ systemd for native installs)

**Summary.** Ollama is configured entirely with environment variables, read when the server starts. In Docker that means `environment:`/`env_file` in compose (or `-e` on `docker run`); apply with a container restart. The authoritative list is `docker exec ollama ollama serve --help`. The tables below are the full working set with defaults (verified 2026-11-03).

## The full table

### Server / network

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_HOST` | `127.0.0.1:11434` (image pre-sets `0.0.0.0:11434`) | listen address; `0.0.0.0:11434` = all interfaces |
| `OLLAMA_ORIGINS` | `127.0.0.1, 0.0.0.0, localhost, ::1` | allowed CORS origins (comma-separated); add `*` or specific hosts |
| `OLLAMA_MAX_BODY_SIZE` | `100M` | max request body size (images, long prompts) |
| `OLLAMA_REGISTRY` | ollama.com registry | default registry host for pull/push |
| `OLLAMA_NO_CLOUD` | `0` | `1` = local-only mode (disables cloud features) |

### Models / memory

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_MODELS` | macOS `~/.ollama/models` · Linux `/usr/share/ollama/.ollama/models` · Docker `/root/.ollama/models` | model store location |
| `OLLAMA_CONTEXT_LENGTH` | `4096` | default context window (tokens) |
| `OLLAMA_KEEP_ALIVE` | `5m` | how long a model stays loaded (`-1` = forever, `0` = unload immediately) |
| `OLLAMA_MAX_LOADED_MODELS` | `3 × GPU count` (3 for CPU) | max concurrently loaded models |
| `OLLAMA_NUM_PARALLEL` | `1` | parallel requests per model (VRAM scales with this × context) |
| `OLLAMA_MAX_QUEUE` | `512` | queued requests before 503 |
| `OLLAMA_NUM_GPU` | `1` | GPU offload: `1` = first GPU, `0` = CPU only, `-1` = all layers to GPU (multi-GPU spread) |
| `OLLAMA_FLASH_ATTENTION` | auto (on when supported) | `1` force on, `0` off |
| `OLLAMA_KV_CACHE_TYPE` | `f16` | KV cache quant: `q8_0` ≈ ½ memory, `q4_0` ≈ ¼ (old name: `OLLAMA_KV_CACHE`) |
| `OLLAMA_GPU_OVERHEAD` | `0` | VRAM bytes reserved per GPU |
| `OLLAMA_VULKAN` | `1` (when GPU devices visible) | `0` disables the Vulkan backend |
| `GGML_VK_VISIBLE_DEVICES` | all | select Vulkan devices (comma-separated ids) |
| `JETSON_JETPACK` | — | `5`/`6` on NVIDIA Jetson |

### Scheduling

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_SCHED_SPREAD` | `0` | allow (0) / prevent (1) partially spreading a model across GPUs |
| `OLLAMA_SCHED_BUSY` | `0` | allow (0) / prevent (1) loading a new model while the GPU is busy |
| `OLLAMA_LLM_LIBRARY` | auto | force a backend library (e.g. `cuda_v12`) |

### Housekeeping

| Variable | Default | Purpose |
|---|---|---|
| `OLLAMA_NOHISTORY` | `0` | `1` = don't save chat history |
| `OLLAMA_NOPRUNE` | `0` | `1` = don't auto-prune unused blobs |
| `OLLAMA_DEBUG` | `0` | `1` = verbose debug logging |
| `OLLAMA_DEBUG_LOG_REQUEST` | `0` | `1` = log request bodies |
| `OLLAMA_LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error`, `panic` |
| `OLLAMA_LOG_FILE` | stderr | log file path |
| `HTTPS_PROXY` | — | outbound proxy for pulls (never set `HTTP_PROXY`) |

## Setting them in Docker

```yaml
# docker-compose.yml
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
      OLLAMA_NUM_GPU: "1"
      OLLAMA_MAX_QUEUE: "256"
      OLLAMA_DEBUG: "1"
```

```bash
docker compose up -d --force-recreate    # env vars only apply at server start
```

`docker run` equivalent: `-e OLLAMA_CONTEXT_LENGTH=8192 -e OLLAMA_KEEP_ALIVE=10m …`.

## Sizing examples

- **24 GB card, one big model, long context:** `OLLAMA_CONTEXT_LENGTH=32768`, `OLLAMA_KV_CACHE_TYPE=q8_0`, `OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_MAX_LOADED_MODELS=1`, `OLLAMA_NUM_PARALLEL=1`.
- **Shared box, small models, low latency:** `OLLAMA_CONTEXT_LENGTH=4096`, `OLLAMA_MAX_LOADED_MODELS=3`, `OLLAMA_NUM_PARALLEL=2`, `OLLAMA_KEEP_ALIVE=-1`.
- **CPU-only:** `OLLAMA_NUM_GPU=0` (or leave it — no GPU is auto-detected), keep the context small.

## systemd (native installs only)

If you're not using Docker and installed natively on Linux, persist env vars with a systemd override:

```bash
sudo systemctl edit ollama
```

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_CONTEXT_LENGTH=8192"
Environment="OLLAMA_KEEP_ALIVE=10m"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
sudo systemctl status ollama
journalctl -e -u ollama          # logs
```

(Override file: `/etc/systemd/system/ollama.service.d/override.conf`.)

## Verifying what's active

```bash
docker exec ollama env | grep OLLAMA        # what the server process actually sees
docker exec ollama ollama ps                # context_length per loaded model
curl -s http://localhost:11434/api/ps | jq  # same, via the API
```

## Sources

- [Ollama FAQ — config, concurrency, KV cache, flash attention](https://docs.ollama.com/faq)
- [Ollama Linux docs — systemd](https://docs.ollama.com/linux)
- `ollama serve --help` (in-container) — the authoritative list

## My experiments

- (fill in: e.g. "set OLLAMA_KV_CACHE_TYPE=q8_0 + FA=1: 27B model at 32K ctx went from OOM to ~2 GB less VRAM in `ollama ps`")
