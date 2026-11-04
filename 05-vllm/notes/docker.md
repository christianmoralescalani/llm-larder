# vLLM in Docker — deep dive

**Summary.** `vllm/vllm-openai` is a complete runtime: vLLM + CUDA/ROCm toolkits + precompiled kernels, entrypoint `vllm serve`. Host needs only the GPU driver (+ Container Toolkit). Two caches must be mounted to make restarts painless: the **HF cache** (`/root/.cache/huggingface`, model weights) and the **compile cache** (`/root/.cache/vllm` = `VLLM_CACHE_ROOT`, torch.compile/Triton artifacts). Config can be passed as CLI args, env-style flags, or a mounted YAML (`--config`).

## Run matrix

```bash
# NVIDIA — the canonical run (detached)
docker run -d --name llm --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -v vllm-cache:/root/.cache/vllm \
  -p 8000:8000 \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen3-0.6B --host 0.0.0.0 --port 8000

# specific GPUs only
docker run --gpus '"device=0,1"' ...

# AMD ROCm
docker run -d --name llm -p 8000:8000 --ipc=host \
  --group-add=video --cap-add=SYS_PTRACE --security-opt seccomp=unconfined \
  --device /dev/kfd --device /dev/dri \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai-rocm:latest \
  --model Qwen/Qwen3-0.6B --host 0.0.0.0 --port 8000

# Intel XPU
docker run -d --name llm -p 8000:8000 --ipc=host --privileged \
  --network=host --device /dev/dri:/dev/dri \
  -v /dev/dri/by-path:/dev/dri/by-path \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai-xpu:latest \
  --model Qwen/Qwen3-0.6B

# interactive shell (debugging, offline inference, bench)
docker run --rm -it --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  --entrypoint /bin/bash vllm/vllm-openai:latest

# Podman
podman run --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 --ipc=host \
  docker.io/vllm/vllm-openai:latest --model Qwen/Qwen3-0.6B
```

## Caches (the two mounts that matter)

| What | Path in container | Mount |
|---|---|---|
| Model weights (HF) | `/root/.cache/huggingface` | `-v ~/.cache/huggingface:/root/.cache/huggingface` |
| Compile cache (torch.compile, Triton, AOT) | `/root/.cache/vllm` (`VLLM_CACHE_ROOT`) | `-v vllm-cache:/root/.cache/vllm` (named volume) |

- Without the HF mount: every container re-downloads the model.
- Without the compile-cache mount: every cold start recompiles (minutes on Hopper+). The cache is invalidated by model/vLLM/version changes — that's expected.
- Gated models: `--env "HF_TOKEN=$HF_TOKEN"`.

## Configuration styles

1. **CLI args** (after the image tag): `--model ... --tensor-parallel-size 2 --gpu-memory-utilization 0.9`.
2. **YAML config file**: `--config config.yaml` (long-form arg names as keys):

   ```yaml
   model: meta-llama/Llama-3.1-8B-Instruct
   host: "0.0.0.0"
   port: 8000
   tensor-parallel-size: 2
   gpu-memory-utilization: 0.9
   ```

   ```bash
   docker run --rm --gpus all -p 8000:8000 --ipc=host \
     -v ~/.cache/huggingface:/root/.cache/huggingface \
     -v "$PWD/config.yaml:/config.yaml:ro" \
     vllm/vllm-openai:latest --config /config.yaml
   ```
3. **Env vars**: `VLLM_API_KEY`, `HF_TOKEN`, `VLLM_ENABLE_CUDA_COMPATIBILITY`, `VLLM_CACHE_ROOT`, … (CLI args win).
4. **vLLM Recipes**: [recipes.vllm.ai](https://recipes.vllm.ai/) converts to `config.yaml` + `env.sh`; mount both and `source env.sh && exec vllm serve --config /recipe/config.yaml`.

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
      - Qwen/Qwen3-0.6B
      - --host
      - 0.0.0.0
      - --port
      - "8000"
      - --enable-prefix-caching
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

## Multi-GPU in Docker

- `--gpus all` (or `'"device=0,1"'`) + `--ipc=host` + `--tensor-parallel-size N` — that's the whole story for single-node TP.
- Multi-node: `--nnodes 2 --node-rank 0|1 --master-addr <ip> --master-port 29501` on each node (see `notes/multi-gpu.md`).

## Non-root

The CUDA image runs as **root** by default; it's also prepared for the built-in `vllm` user (UID 2000, GID 0):

```bash
docker run --rm --gpus all --user 2000:0 -p 8000:8000 \
  -v ~/.cache/huggingface:/home/vllm/.cache/huggingface \
  vllm/vllm-openai:latest meta-llama/Llama-3.1-8B-Instruct
```

With `--user 2000:0`, mount writable paths under `/home/vllm` (not `/root`).

## Custom images

```dockerfile
# optional deps (e.g. audio) — match the vLLM version to the base tag!
FROM vllm/vllm-openai:v0.32.0
RUN uv pip install --system vllm[audio]==0.32.0

# bleeding-edge transformers (new model not in the pinned release)
FROM vllm/vllm-openai:latest
RUN uv pip install --system git+https://github.com/huggingface/transformers.git
```

Build from source: `DOCKER_BUILDKIT=1 docker build . --target vllm-openai --tag vllm/vllm-openai --file docker/Dockerfile` (from a repo clone; `--build-arg VLLM_USE_PRECOMPILED=1` skips kernel compilation).

## Gotchas

1. **`--ipc=host`** — mandatory for TP; otherwise worker processes can't share memory.
2. **Port 8000 inside the container** — the server binds `--host 0.0.0.0 --port 8000` by default; publish with `-p 8000:8000`.
3. **`latest` tag** — fine for experimenting; pin `v<version>` for anything you rely on.
4. **First-request latency** — the model loads + CUDA graphs capture at startup (watch `docker logs` until `Application startup complete`).
5. **No auth by default** — `--api-key`/`VLLM_API_KEY` only guards `/v1`, `/v2`, `/inference` paths (not `/invocations`); don't expose publicly without a reverse proxy.
6. **Image size** — several GB (CUDA + kernels); `docker system df` after a few tag pulls.
7. **WSL not project-verified** — native Linux is the tested Docker path.

## Sources

- [Using Docker](https://docs.vllm.ai/en/stable/deployment/docker/)
- [Server Arguments (YAML config)](https://docs.vllm.ai/en/stable/configuration/serve_args/)
- [Faster startup (compile cache)](https://docs.vllm.ai/en/stable/configuration/optimization/#faster-startup)
- [vLLM Recipes](https://recipes.vllm.ai/)

## My experiments

- (fill in: e.g. "compose file with both cache mounts: first boot X min (model download + compile), second boot Y s; TP=2 on 2×24GB with `--ipc=host` served 70B-AWQ at Z t/s")
