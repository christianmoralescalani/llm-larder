# Cheatsheet — Ollama in Docker

**TL;DR.** Official image `ollama/ollama` (Docker Hub); models in a volume at `/root/.ollama`; port 11434; GPU via `--gpus all` (NVIDIA) or `:rocm` + `--device` (AMD). The image pre-sets `OLLAMA_HOST=0.0.0.0:11434`, so `-p 11434:11434` works out of the box. Verified on `ollama/ollama:0.40.14`, 2026-11-03.

## One-liners

```bash
# CPU
docker run -d --name ollama -v ollama:/root/.ollama -p 11434:11434 ollama/ollama

# NVIDIA GPU (host needs NVIDIA Container Toolkit)
docker run -d --name ollama --gpus all -v ollama:/root/.ollama -p 11434:11434 ollama/ollama

# AMD GPU
docker run -d --name ollama --device /dev/kfd --device /dev/dri \
  -v ollama:/root/.ollama -p 11434:11434 ollama/ollama:rocm

# Vulkan (standard image, AMD/Intel)
docker run -d --name ollama --device /dev/kfd --device /dev/dri \
  -v ollama:/root/.ollama -p 11434:11434 ollama/ollama

# with your own GGUFs
docker run -d --name ollama --gpus all \
  -v ollama:/root/.ollama -v ~/Downloads/models:/models:ro \
  -p 11434:11434 ollama/ollama
```

## Compose (canonical)

```yaml
services:
  ollama:
    image: ollama/ollama:0.40.14
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
      # - ~/Downloads/models:/models:ro
    environment:
      OLLAMA_CONTEXT_LENGTH: "8192"
      OLLAMA_KEEP_ALIVE: "10m"
      OLLAMA_KV_CACHE_TYPE: "q8_0"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 30s
      timeout: 10s
      retries: 3
volumes:
  ollama_data:
```

## Daily ops

```bash
docker compose up -d && docker compose logs -f ollama
docker exec -it ollama ollama run llama3.2
docker exec ollama ollama ps
curl -s http://localhost:11434/api/version
docker compose pull && docker compose up -d    # update (models survive — they're in the volume)
docker compose down && docker volume rm <proj>_ollama_data   # uninstall (⚠ deletes models)
```

## Tags

| Tag | Notes |
|---|---|
| `latest` | stable (0.40.14), amd64+arm64, ~3.2 GB compressed |
| `0.40.14` | pinned |
| `rocm` | AMD (amd64 only, ~1.4 GB) |
| `0.40.15-rc0` | pre-release |

## Gotchas

- The image runs as **root** → models at `/root/.ollama`.
- `OLLAMA_HOST` is pre-set to `0.0.0.0:11434` — don't override it to `127.0.0.1` if you use `-p`.
- File paths in `ollama run`/Modelfile are **container paths** — mount the directory.
- No built-in auth — firewall the port, don't expose to the internet.
- Docker Desktop on macOS: CPU only (no GPU passthrough).

## Verified on

- [Ollama Docker docs](https://docs.ollama.com/docker) + Docker Hub `ollama/ollama`, 2026-11-03.
