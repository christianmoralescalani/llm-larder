# Ollama in Docker — deep dive

**Summary.** The official `ollama/ollama` image is a complete runtime: the `ollama` binary plus llama.cpp backends (CUDA libraries bundled, Vulkan bundled), running `ollama serve` as PID 1 as root. This note covers GPU passthrough, where models live, exposing the API, persistence, multiple instances, and the gotchas (the `OLLAMA_HOST` default, container-side paths, no built-in auth).

## What's in the image

- `ollama` binary (CLI + server in one).
- llama.cpp inference backends: CUDA (bundled — you only need the host *driver*, not a CUDA toolkit), Vulkan (bundled, auto-enabled when GPU devices are visible), ROCm (separate `rocm` tag).
- Runs as **root**; models at **`/root/.ollama`** (the directory you mount).
- Pre-sets `OLLAMA_HOST=0.0.0.0:11434`, so `-p 11434:11434` works out of the box.

## GPU support

### NVIDIA

Host prerequisites: NVIDIA driver + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) (`nvidia-ctk runtime configure --runtime=docker && systemctl restart docker`).

```bash
docker run -d --name ollama --gpus all -v ollama:/root/.ollama -p 11434:11434 ollama/ollama
```

Multi-GPU: `--gpus all` passes every card; Ollama loads a model on a single GPU when it fits, otherwise spreads it across all GPUs. `--gpus '"device=0,1"'` selects specific cards.

### AMD (ROCm)

```bash
docker run -d --name ollama --device /dev/kfd --device /dev/dri \
  -v ollama:/root/.ollama -p 11434:11434 ollama/ollama:rocm
```

### Vulkan

The standard image includes Vulkan; it activates when the container can see the GPU device:

```bash
docker run -d --name ollama --device /dev/kfd --device /dev/dri \
  -v ollama:/root/.ollama -p 11434:11434 ollama/ollama
```

`OLLAMA_VULKAN=0` disables it; `GGML_VK_VISIBLE_DEVICES=0` selects a specific device.

### NVIDIA Jetson

`docker run … -e JETSON_JETPACK=5` (or `6`) — the container can't auto-detect the JetPack version.

### Docker Desktop on macOS

No GPU passthrough — CPU only.

## Model storage

- Default inside the container: `/root/.ollama/models` (content-addressed `blobs/sha256-…`).
- Mount options:
  - **Named volume** (official default): `-v ollama:/root/.ollama` — simple, Docker-managed.
  - **Bind mount** (what I use, matches this repo's llama.cpp setup): `-v ~/Downloads/models-ollama:/root/.ollama` — models visible from the host, easy to back up, shareable with other tools.
- Move the store: `OLLAMA_MODELS=/models` + mount a directory there.
- Disk: budget the *uncompressed* model size (8B Q4_K_M ≈ 5 GB) plus headroom — pulls download to a temp file before linking into the store.

## Using your own GGUFs / safetensors

```bash
docker run -d --name ollama --gpus all \
  -v ollama:/root/.ollama \
  -v ~/Downloads/models:/models:ro \
  -p 11434:11434 ollama/ollama
```

Then build a model from a GGUF (full workflow in [modelfile.md](modelfile.md)):

```bash
docker exec ollama ollama create my-qwen -f /models/Modelfile-qwen
# Modelfile-qwen:
#   FROM /models/Qwen3.8-27B-UD-Q4_K_XL.gguf
```

The path in `FROM` is resolved **inside the container** — the file must be on a mounted volume.

## Exposing the API

- Local: `-p 11434:11434` → `http://localhost:11434` on the host.
- LAN: the image already binds `0.0.0.0` inside the container; the host `-p` mapping exposes it. Firewall it. **Ollama has no built-in auth** — anyone who can reach the port can run your models (and use your registry keys if you've signed in). Don't expose 11434 to the internet; put an authed reverse proxy in front.
- Different host port: `-p 1234:11434` → `http://localhost:1234`.
- Container-to-container: on the same compose network, use `http://ollama:11434` (service name) — no host port needed.

## Networking modes

| Mode | When |
|---|---|
| default bridge + `-p` (recommended) | normal setup |
| `--network host` | no port mapping; server binds `OLLAMA_HOST` directly on the host (Linux) |
| shared compose network | frontends (Open WebUI, …) talk to `http://ollama:11434` |

## Environment variables in compose

```yaml
    environment:
      OLLAMA_HOST: "0.0.0.0:11434"
      OLLAMA_CONTEXT_LENGTH: "8192"
      OLLAMA_KEEP_ALIVE: "10m"
      OLLAMA_KV_CACHE_TYPE: "q8_0"
      OLLAMA_FLASH_ATTENTION: "1"
      OLLAMA_NUM_PARALLEL: "1"
      OLLAMA_MAX_LOADED_MODELS: "1"
      OLLAMA_DEBUG: "1"
```

Or `env_file: ./ollama.env`. Full table with defaults: [configuration.md](configuration.md). Changes only apply at server start: `docker compose up -d --force-recreate`.

## Multiple instances

```yaml
services:
  ollama-small:
    image: ollama/ollama:0.40.14
    ports: ["11434:11434"]
    volumes: [ollama_small:/root/.ollama]
  ollama-big:
    image: ollama/ollama:0.40.14
    ports: ["11435:11434"]
    volumes: [ollama_big:/root/.ollama]
```

Each instance = its own model store + port. Useful for "fast small model + big model" boxes.

## Resource limits

- GPU: `--gpus all` (or specific devices).
- RAM/CPU caps (compose): `mem_limit: 32g`, `cpus: "8"` — useful when sharing the box.
- VRAM can't be capped from Docker; control it with `OLLAMA_NUM_PARALLEL`, `OLLAMA_CONTEXT_LENGTH`, `OLLAMA_MAX_LOADED_MODELS`.

## Healthcheck, restart, logs

```yaml
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 30s
      timeout: 10s
      retries: 3
```

```bash
docker logs -f ollama                # server log (startup, model loads, per-request)
docker exec ollama ollama ps         # what's loaded, VRAM, GPU/CPU split
```

`OLLAMA_DEBUG=1` + `OLLAMA_LOG_LEVEL=debug` for verbose logs (scheduler decisions, layer offload).

## Proxy (corporate networks)

```bash
docker run … -e HTTPS_PROXY=https://proxy.example.com:8080 ollama/ollama
```

Self-signed corporate CA: bake it into the image (from the official FAQ):

```dockerfile
FROM ollama/ollama
COPY my-ca.pem /usr/local/share/ca-certificates/my-ca.crt
RUN update-ca-certificates
```

Use `HTTPS_PROXY` only — setting `HTTP_PROXY` can break client connections.

## Gotchas

1. **`OLLAMA_HOST`** — the image defaults to `0.0.0.0:11434`; overriding it to `127.0.0.1:11434` breaks host port mapping (server binds the container's loopback).
2. **Paths are container paths** — `ollama run gemma3 ./img.png` resolves `./img.png` *inside* the container. Mount the directory or `docker cp` the file first.
3. **No auth** — the API is open to whoever can reach the port.
4. **Container runs as root** — if Ollama can't write to a bind-mounted `/root/.ollama`, fix ownership on the host dir (or use a named volume).
5. **GPU not visible** → `docker logs ollama` shows no CUDA device; check `nvidia-smi` on the host and the container toolkit install.
6. **Killed mid-pull** → partial blob; re-run `ollama pull` to resume.

## Sources

- [Ollama Docker docs](https://docs.ollama.com/docker)
- [Ollama FAQ — proxy, GPU in Docker, model storage](https://docs.ollama.com/faq)
- [NVIDIA Container Toolkit install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

## My experiments

- (fill in: e.g. "bind-mounted ~/Downloads/models-ollama; imported my Qwen3.8-27B GGUF with a one-line Modelfile; `ollama ps` → 100% GPU")
