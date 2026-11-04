# llama.cpp in Docker — deep dive

**Summary.** The official `ghcr.io/ggml-org/llama.cpp` images are a complete runtime: prebuilt binaries at `/app/llama-*`, no host toolchain needed. For GPUs you only need the host *driver* (NVIDIA driver + Container Toolkit for CUDA; device nodes for ROCm/Vulkan/SYCL). Models live wherever you mount (convention: `/models`); `-hf` downloads go to the HF cache inside the container. The two gotchas that bite everyone: the server listens on **127.0.0.1:9931** by default (so `-p 8080:8080` silently does nothing until you pass `--host 0.0.0.0 --port 8080`), and GPU images are built but **not CI-tested** beyond compiling.

## Image matrix

Three flavors × backends (platforms in parentheses):

| Flavor | CPU | CUDA 12 | CUDA 13 | ROCm | Vulkan | SYCL (Intel) | MUSA | OpenVINO |
|---|---|---|---|---|---|---|---|---|
| `full` (all tools + Python) | `:full` | `:full-cuda` | `:full-cuda13` | `:full-rocm` | `:full-vulkan` | `:full-intel` | `:full-musa` | `:full-openvino` |
| `light` (cli + completion) | `:light` | `:light-cuda` | `:light-cuda13` | `:light-rocm` | `:light-vulkan` | `:light-intel` | `:light-musa` | `:light-openvino` |
| `server` (llama-server only) | `:server` | `:server-cuda` | `:server-cuda13` | `:server-rocm` | `:server-vulkan` | `:server-intel` | `:server-musa` | `:server-openvino` |

- Base platforms: `linux/amd64`, `linux/arm64`, `linux/s390x` (plus `:full-s390x` etc. aliases).
- `full` = quantize/convert/bench everything; `light` = just chat; `server` = just serve. Pick `server*` for a lean serving container.
- GPU images are not CI-tested beyond being built — if one misbehaves, build locally from `.devops/*.Dockerfile` (see below).

## Quick start

```bash
# CPU server
docker run -p 8080:8080 -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server \
  --model /models/model.gguf --host 0.0.0.0 --port 8080

# NVIDIA GPU (host: NVIDIA driver + nvidia-container-toolkit)
docker run -p 8080:8080 --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --host 0.0.0.0 --port 8080 --n-gpu-layers -1

# interactive CLI (light image; llama-cli is the default entrypoint)
docker run --rm --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  --model /models/model.gguf --n-gpu-layers -1

# pull a model straight from Hugging Face (no local file needed)
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF
```

## GPU support

### NVIDIA (CUDA)

Host prerequisites: NVIDIA driver + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) (`nvidia-ctk runtime configure --runtime=docker && systemctl restart docker`). Then `--gpus all` (or `--gpus '"device=0,1"'` for a subset). You do **not** need the CUDA toolkit on the host — it's in the image. Pick `:server-cuda` (CUDA 12) or `:server-cuda13` (CUDA 13) to match your driver generation.

### AMD (ROCm)

`--device /dev/kfd --device /dev/dri` with the `:server-rocm` image (amd64 only).

### AMD/Intel (Vulkan)

`--device /dev/dri` with the `:server-vulkan` image — the universal fallback when CUDA/ROCm images don't fit.

### Intel (SYCL)

`--device /dev/dri/renderD128 --device /dev/dri/card0` with the `:server-intel` image; Intel GPU driver on the host.

## Where models live

- **Local files**: mount a host dir (`-v ~/Downloads/models:/models`) and pass container paths to `--model`.
- **Hugging Face**: `-hf <user>/<repo>[:quant]` downloads on demand (quant defaults to `Q4_K_M`, mmproj auto-downloaded if present). Files land in the HF cache — mount a volume at the cache dir (or set `HF_HOME`) if you want them to survive container recreation. `--offline` forces cache-only.
- **Private repos**: `-hft <token>` or the `HF_TOKEN` env var.

## Compose (canonical)

```yaml
services:
  llamacpp:
    image: ghcr.io/ggml-org/llama.cpp:server-cuda
    ports:
      - "8080:8080"
    volumes:
      - ~/Downloads/models:/models
    environment:
      LLAMA_ARG_MODEL: /models/model.gguf
      LLAMA_ARG_CTX_SIZE: "32768"
      LLAMA_ARG_N_PARALLEL: "2"
      LLAMA_ARG_PORT: "8080"
      LLAMA_ARG_ENDPOINT_METRICS: "1"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

Every CLI flag has a `LLAMA_ARG_*` env var (e.g. `--ctx-size` → `LLAMA_ARG_CTX_SIZE`, `--api-key` → `LLAMA_API_KEY`) — see `cheatsheets/llama-server-flags.md`. CLI args override env vars.

## Persistence

- Models: bind mount or named volume (above).
- HF cache: mount a volume (e.g. `-v hf-cache:/root/.cache/huggingface`).
- Slot KV cache: `--slot-save-path /models/slot-cache` + `POST /slots/{id}?action=save|restore` (this is what replaced the old `llama-save-load-state` tool).

## Building images locally

Needed only for exotic CUDA/ROCm versions:

```bash
git clone https://github.com/ggml-org/llama.cpp && cd llama.cpp
docker build -t local/llama.cpp:server-cuda --target server -f .devops/cuda.Dockerfile .
# defaults: CUDA_VERSION=12.8.1, all GPU archs; override with --build-arg
```

## Gotchas

1. **Default bind is 127.0.0.1:9931** — inside the container that's unreachable from the host. Always pass `--host 0.0.0.0 --port 8080` (or the env vars) when publishing a port.
2. **`--n-gpu-layers`** — defaults to `auto`, but `--n-gpu-layers -1` (all layers) is the usual GPU setting; use `--fit off` if auto-fit fights you.
3. **GPU images aren't CI-tested** — a broken `:server-cuda` release is possible; pin tags and verify with `llama-server --list-devices` / `docker logs`.
4. **No auth by default** — `--api-key` is optional; don't expose the port to the internet without it (plus a reverse proxy).
5. **WSL not verified** by the project for Docker; native Linux is the tested path.
6. **`full` image + Python** — quantization needs the Python deps; the image ships the tools, `pip install -r requirements.txt` if a conversion script complains about a missing package (or `uv pip install ...`).

## Sources

- [llama.cpp Docker docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/docker.md)
- [Server README — env vars & compose example](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [NVIDIA Container Toolkit install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

## My experiments

- (fill in: e.g. "`:server-cuda` + `--gpus all` + Qwen3.8-27B Q4_K_XL with MTP draft: loads in X s, Y tok/s decode, web UI at :8080; `--flash-attn on --cache-type-k q4_0` cut KV VRAM by ~Z%")
