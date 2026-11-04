# Installing Ollama (Docker-first)

**Summary.** Ollama is a single-binary runtime built on llama.cpp that bundles a model-registry client, a chat CLI, and an HTTP API. In this repo we run it in Docker: the official image `ollama/ollama` on Docker Hub is the whole story — no host install needed. The container runs the server (`ollama serve`) as PID 1; you talk to it with `curl` (REST API) or `docker exec ollama ollama …` (CLI). Models live in a volume at `/root/.ollama`, so they survive image updates and reinstalls.

## Prerequisites

- Docker Engine (or Docker Desktop).
- **NVIDIA GPU (optional):** host needs the NVIDIA driver + the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) so `docker run --gpus all` works. Verify with `nvidia-smi` on the host.
- **AMD GPU (optional):** nothing extra to install — pass `--device /dev/kfd --device /dev/dri` and use the `rocm` image tag.
- Disk: ~3 GB for the image (compressed) + whatever models you pull (see [02-hardware-and-vram](../../02-hardware-and-vram/notes/vram-formula.md)).

## Quickstart — `docker run`

### CPU only

```bash
docker run -d \
  --name ollama \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama
```

### NVIDIA GPU

```bash
docker run -d \
  --name ollama \
  --gpus all \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama
```

### AMD GPU (ROCm)

```bash
docker run -d \
  --name ollama \
  --device /dev/kfd --device /dev/dri \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  ollama/ollama:rocm
```

`-v ollama:/root/.ollama` is a named volume holding everything Ollama writes (models, registry keys, history). The container runs as root, hence `/root/.ollama`.

## Quickstart — docker compose (recommended)

```yaml
# docker-compose.yml
services:
  ollama:
    image: ollama/ollama:0.40.14        # pin the version for reproducibility
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama        # models persist across image updates
      # - ~/Downloads/models:/models:ro  # optional: your own GGUFs (see notes/docker.md)
    environment:
      - OLLAMA_CONTEXT_LENGTH=8192
      - OLLAMA_KEEP_ALIVE=10m
    # NVIDIA GPU: uncomment (remove on CPU-only hosts)
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
volumes:
  ollama_data:
```

```bash
docker compose up -d
```

Notes:

- The image pre-sets `OLLAMA_HOST=0.0.0.0:11434` internally, so the `-p 11434:11434` mapping works out of the box. If you override `OLLAMA_HOST`, keep `0.0.0.0` or the host mapping stops working (see [docker.md](docker.md)).
- Remove the `deploy.resources` block on a CPU-only host — without the NVIDIA toolkit the compose file fails to start.

## Verify it's working

```bash
docker logs ollama | tail                 # "Listening on 0.0.0.0:11434"
curl http://localhost:11434/api/version   # {"version":"0.40.14"}
docker exec -it ollama ollama -v          # client version
docker exec -it ollama ollama run llama3.2 "say hi in one word"
```

The first `run` pulls the model (a few GB) — watch progress in `docker logs ollama`.

## Image tags (Docker Hub, verified 2026-11-03)

| Tag | What it is |
|---|---|
| `ollama/ollama:latest` | current stable (0.40.14 at writing), linux/amd64 + linux/arm64, ~3.2 GB compressed |
| `ollama/ollama:0.40.14` | version pin — use this in compose for reproducibility |
| `ollama/ollama:0.40.15-rc0` | pre-release |
| `ollama/ollama:rocm` | AMD GPU build (linux/amd64 only, ~1.4 GB compressed) |
| `ollama/ollama:0.40.14-rocm` | pinned ROCm build |

Vulkan is bundled in the standard image (auto-enabled when the container sees GPU devices) — see [docker.md](docker.md).

## Native install (reference only)

This repo standardizes on Docker. If you ever run natively:

- **Linux:** `curl -fsSL https://ollama.com/install.sh | sh` — installs to `/usr` + a systemd service (`ollama.service`, user `ollama`, models in `/usr/share/ollama/.ollama/models`). Pin a version with `OLLAMA_VERSION=x.y.z` before the script.
- **macOS:** `curl -fsSL https://ollama.com/install.sh | sh` or the `.dmg` from ollama.com/download.
- **Windows:** `irm https://ollama.com/install.ps1 | iex` or `OllamaSetup.exe`.
- **Updating (Linux):** re-run the install script. **Uninstall:** stop/disable the service, remove the unit file, binary, libs, then `userdel ollama` (details in the [Linux docs](https://docs.ollama.com/linux)).

## Updating (Docker)

```bash
docker compose pull && docker compose up -d   # new image, same volume → models intact
docker image prune                             # drop the old image
```

Models live in the volume, not the image — updates never touch them. If you pinned a version, bump the tag first.

## Uninstalling (Docker)

```bash
docker compose down                              # stop + remove container
docker volume rm <project>_ollama_data           # ⚠ deletes ALL models
docker rmi ollama/ollama:0.40.14
```

`docker volume ls` to find the volume name. Skip the volume removal if you want to keep your models.

## Sources

- [Ollama Docker docs](https://docs.ollama.com/docker) — official CPU/NVIDIA/AMD/Vulkan run commands
- [Docker Hub: ollama/ollama](https://hub.docker.com/r/ollama/ollama) — tags, sizes, architectures
- [Ollama Linux docs](https://docs.ollama.com/linux) — native install, systemd, uninstall

## My experiments

- (fill in: e.g. "compose up on my 24 GB card: first `ollama run qwen3` pulled ~18 GB in ~4 min; `ollama ps` shows 100% GPU")
