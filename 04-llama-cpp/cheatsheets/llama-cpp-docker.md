# Cheatsheet — llama.cpp Docker

**TL;DR.** Image = `ghcr.io/ggml-org/llama.cpp:{full|light|server}[-cuda|-cuda13|-rocm|-vulkan|-intel|-musa|-openvino]`. Host needs only the GPU **driver** (+ NVIDIA Container Toolkit for CUDA). Models: mount at `/models` or `-hf`. Server default bind is **127.0.0.1:9931** → always `--host 0.0.0.0 --port 8080` when publishing. Every flag = `LLAMA_ARG_*` env var. Verified against llama.cpp master (v0.8.0, 2026-10-28), 2026-11-03.

## Image matrix

| Flavor | CPU | CUDA 12 | CUDA 13 | ROCm | Vulkan | SYCL | MUSA | OpenVINO |
|---|---|---|---|---|---|---|---|---|
| `full` (all tools + Python) | `:full` | `:full-cuda` | `:full-cuda13` | `:full-rocm` | `:full-vulkan` | `:full-intel` | `:full-musa` | `:full-openvino` |
| `light` (cli + completion) | `:light` | `:light-cuda` | `:light-cuda13` | `:light-rocm` | `:light-vulkan` | `:light-intel` | `:light-musa` | `:light-openvino` |
| `server` (llama-server only) | `:server` | `:server-cuda` | `:server-cuda13` | `:server-rocm` | `:server-vulkan` | `:server-intel` | `:server-musa` | `:server-openvino` |

Platforms: `linux/amd64`, `linux/arm64`, `linux/s390x`. Nightlies: `b*` tags. GPU images are built but **not CI-tested** beyond compiling.

## Run commands

```bash
# CPU server
docker run -p 8080:8080 -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server \
  --model /models/model.gguf --host 0.0.0.0 --port 8080

# NVIDIA GPU (host: driver + nvidia-container-toolkit)
docker run -p 8080:8080 --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --host 0.0.0.0 --port 8080 --n-gpu-layers -1

# specific GPUs
docker run --gpus '"device=0,1"' ...

# interactive CLI
docker run --rm --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  --model /models/model.gguf --n-gpu-layers -1

# Hugging Face pull (no local file)
docker run --rm ghcr.io/ggml-org/llama.cpp:light -hf ggml-org/Qwen3.5-0.8B-GGUF

# AMD ROCm
docker run -p 8080:8080 --device /dev/kfd --device /dev/dri \
  -v ~/Downloads/models:/models ghcr.io/ggml-org/llama.cpp:server-rocm \
  --model /models/model.gguf --host 0.0.0.0 --port 8080

# Vulkan (AMD/Intel fallback)
docker run -p 8080:8080 --device /dev/dri \
  -v ~/Downloads/models:/models ghcr.io/ggml-org/llama.cpp:server-vulkan \
  --model /models/model.gguf --host 0.0.0.0 --port 8080

# Intel SYCL
docker run -p 8080:8080 --device /dev/dri/renderD128 --device /dev/dri/card0 \
  -v ~/Downloads/models:/models ghcr.io/ggml-org/llama.cpp:server-intel \
  --model /models/model.gguf --host 0.0.0.0 --port 8080

# quantize (full image)
docker run --rm -v ~/Downloads/models:/m ghcr.io/ggml-org/llama.cpp:full \
  llama-quantize /m/model-bf16.gguf /m/model-Q4_K_M.gguf Q4_K_M

# benchmark
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda llama-bench -m /m/model.gguf
```

## Compose (env-var style)

```yaml
services:
  llamacpp:
    image: ghcr.io/ggml-org/llama.cpp:server-cuda
    ports: ["8080:8080"]
    volumes:
      - ~/Downloads/models:/models
      - hf-cache:/root/.cache/huggingface
    environment:
      LLAMA_ARG_MODEL: /models/model.gguf
      LLAMA_ARG_HOST: 0.0.0.0
      LLAMA_ARG_PORT: "8080"
      LLAMA_ARG_N_GPU_LAYERS: "-1"
      LLAMA_ARG_CTX_SIZE: "32768"
      LLAMA_ARG_N_PARALLEL: "2"
      LLAMA_ARG_FLASH_ATTN: "on"
      LLAMA_ARG_CACHE_TYPE_K: q4_0
      LLAMA_ARG_CACHE_TYPE_V: q4_0
      LLAMA_ARG_ENDPOINT_METRICS: "1"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
volumes:
  hf-cache:
```

## Env var mapping

`--flag-name` → `LLAMA_ARG_FLAG_NAME`. Exceptions: `--api-key` → `LLAMA_API_KEY`, `--hf-token` → `HF_TOKEN`, `--mmproj-device` → `MTMD_BACKEND_DEVICE`. Booleans: `1/0, on/off, true/false, enabled/disabled`. CLI args override env vars.

## Gotchas

1. **127.0.0.1:9931 default** — port publishing silently no-ops without `--host 0.0.0.0 --port 8080`.
2. **No CUDA toolkit needed on host** — it's in the image; driver + Container Toolkit only.
3. **GPU images not CI-tested** — pin tags, verify with `docker logs` / `--list-devices`.
4. **HF cache is ephemeral** — mount a volume at `/root/.cache/huggingface` (or set `HF_HOME`) to keep `-hf` downloads.
5. **`full` image for Python tools** — `convert_hf_to_gguf.py` needs the Python deps (in the image); `pip install -U transformers` if a new model needs transformers 5.
6. **No auth by default** — `--api-key` before exposing beyond localhost.
7. **WSL not project-verified** — native Linux is the tested Docker path.
8. **Local builds** for exotic CUDA/ROCm: `docker build --target server -f .devops/cuda.Dockerfile .` (from a repo clone).
