# Cheatsheet — vLLM Docker

**TL;DR.** One image: `vllm/vllm-openai` (NVIDIA) — entrypoint is `vllm serve`, so everything after the tag is a serve flag. Host needs only the GPU driver + Container Toolkit. Two mounts make restarts painless: HF cache (weights) and `~/.cache/vllm` (compile cache). `--ipc=host` is mandatory for tensor parallel. ROCm: `vllm/vllm-openai-rocm`; Intel: `vllm/vllm-openai-xpu`. Verified against vLLM v0.32.0 (stable, 2026-10-18), 2026-11-03.

## Images & tags

| Image | Backend | Tags |
|---|---|---|
| `vllm/vllm-openai` | NVIDIA CUDA (Ubuntu 24.04) | `latest`, `v<version>`, `latest-x86_64`/`latest-aarch64`, `v<ver>-x86_64`/`-aarch64` |
| `vllm/vllm-openai-rocm` | AMD ROCm | `latest` (ROCm 10.0), `nightly` |
| `vllm/vllm-openai-xpu` | Intel XPU | `latest`, `nightly` |

- `-ubuntu2404` suffixes are **no longer published** — use `latest`/`v<version>`.
- Pin `v<version>` in production.
- Optional deps (audio etc.) not included → custom image.

## Run commands

```bash
# NVIDIA — canonical (detached)
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -v vllm-cache:/root/.cache/vllm \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-7B-Instruct --host 0.0.0.0 --port 8000

# specific GPUs
docker run --gpus '"device=0,1"' ...

# AMD ROCm
docker run -d --name llm -p 8000:8000 --ipc=host \
  --group-add=video --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
  --device /dev/kfd --device /dev/dri \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai-rocm:latest \
  --model Qwen/Qwen2.5-7B-Instruct --host 0.0.0.0 --port 8000

# Intel XPU
docker run -d --name llm -p 8000:8000 --ipc=host --privileged \
  --network=host --device /dev/dri:/dev/dri \
  -v /dev/dri/by-path:/dev/dri/by-path \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai-xpu:latest --model Qwen/Qwen2.5-7B-Instruct

# interactive shell (offline inference, bench, debugging)
docker run --rm -it --gpus all --entrypoint /bin/bash \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest

# run a Python script (offline LLM)
docker run --rm --gpus all --entrypoint python \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -v /tmp/gen.py:/gen.py:ro \
  vllm/vllm-openai:latest /gen.py

# vllm client tools (chat/complete/bench) against a running server
docker run --rm --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest chat --url http://localhost:8000/v1

# Podman
podman run --device nvidia.com/gpu=all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  docker.io/vllm/vllm-openai:latest --model Qwen/Qwen2.5-7B-Instruct
```

## Caches (mount both)

| What | Container path | Mount |
|---|---|---|
| HF weights | `/root/.cache/huggingface` | `-v ~/.cache/huggingface:/root/.cache/huggingface` |
| Compile cache (torch.compile/Triton/AOT) | `/root/.cache/vllm` | `-v vllm-cache:/root/.cache/vllm` |

Gated models: `--env "HF_TOKEN=$HF_TOKEN"`.

## Compose (canonical)

```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    ports: ["8000:8000"]
    volumes:
      - ~/.cache/huggingface:/root/.cache/huggingface
      - vllm-cache:/root/.cache/vllm
    environment:
      HF_TOKEN: ${HF_TOKEN}
      VLLM_API_KEY: ${VLLM_API_KEY:-}
    command:
      - --model
      - Qwen/Qwen2.5-7B-Instruct
      - --host
      - 0.0.0.0
      - --port
      - "8000"
      - --max-model-len
      - "32k"
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
volumes:
  vllm-cache:
```

## Config styles

1. CLI args after the image tag.
2. `--config /config.yaml` (long-form names; mount the file read-only).
3. Env vars: `VLLM_API_KEY`, `HF_TOKEN`, `VLLM_ENABLE_CUDA_COMPATIBILITY`, `VLLM_CACHE_ROOT` (CLI wins).
4. vLLM Recipes → `config.yaml` + `env.sh` (recipes.vllm.ai).

## Non-root & custom images

```bash
# built-in vllm user (UID 2000, GID 0) — mount under /home/vllm
docker run --rm --gpus all --user 2000:0 -p 8000:8000 \
  -v ~/.cache/huggingface:/home/vllm/.cache/huggingface \
  vllm/vllm-openai:latest Qwen/Qwen2.5-7B-Instruct
```

```dockerfile
# optional deps — match the version to the base tag
FROM vllm/vllm-openai:v0.32.0
RUN uv pip install --system vllm[audio]==0.32.0
```

## Multi-GPU

- Single-node TP: `--gpus all` + `--ipc=host` + `--tensor-parallel-size N`. Nothing else.
- Multi-node: same + `--nnodes 2 --node-rank 0|1 --master-addr <ip> --master-port 29501` per node.

## Gotchas

- **`--ipc=host`** or TP crashes (shared memory).
- Server binds `0.0.0.0:8000` inside the container → `-p 8000:8000`.
- First boot = model download + compile (minutes); second boot with both caches = fast.
- `latest` follows newest release — pin `v<version>` for stability.
- No auth by default; `/metrics` + `/invocations` bypass `--api-key`.
- Old driver + new CUDA image: `VLLM_ENABLE_CUDA_COMPATIBILITY=1` (datacenter GPUs).
- Image is several GB — `docker system df`.
