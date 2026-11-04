# Installing vLLM — Docker-first

**Summary.** In this repo the install is the official Docker image `vllm/vllm-openai` on Docker Hub — no Python, no CUDA toolkit, no pip on the host. The image's entrypoint *is* `vllm serve`, so `docker run vllm/vllm-openai:latest <model>` starts an OpenAI-compatible server. pip/conda installs exist for reference (native dev work, `vllm bench` on the host) but are not the examples here. Verified against vLLM v0.32.0 (stable, 2026-10-18; v0.33.0rc3 published 2026-11-02), docs.vllm.ai, 2026-11-03.

## The image

| Image | Platform | Notes |
|---|---|---|
| `vllm/vllm-openai` | NVIDIA CUDA | the default; Ubuntu 24.04 base; tags `latest`, `v<version>`, `latest-x86_64` / `latest-aarch64` (also `v<version>-x86_64`/`-aarch64`) |
| `vllm/vllm-openai-rocm` | AMD ROCm | `latest` = ROCm 10.0, `nightly` = dev branch; legacy `-rocm72` tags kept for transition. AMD's old `rocm/vllm` images are **deprecated** |
| `vllm/vllm-openai-xpu` | Intel XPU | stable since v0.26.0, plus `nightly` |

- Older `-ubuntu2404` tag suffixes are **no longer published** — use `latest` / `v<version>`.
- Pin `v<version>` (e.g. `v0.32.0`) in production; `latest` tracks the newest release.
- The image ships CUDA 12/13 toolkits and precompiled kernels; the host needs only the **driver** + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).
- Optional dependencies (e.g. `vllm[audio]`) are *not* included (licensing) — derive an image if you need them (see `notes/docker.md`).

## Quick start (Docker, NVIDIA)

```bash
# host: nvidia driver + container toolkit (nvidia-ctk runtime configure --runtime=docker && systemctl restart docker)

docker run --rm --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen3-0.6B
```

Then `curl http://localhost:8000/v1/models`. `--ipc=host` (or `--shm-size`) is required for multi-GPU/tensor parallel — PyTorch shares data between worker processes via shared memory.

## Hardware requirements

| Backend | What you need |
|---|---|
| **NVIDIA CUDA** (primary) | driver + Container Toolkit; CUDA 13 images need R580+ driver (or `VLLM_ENABLE_CUDA_COMPATIBILITY=1` for R535/R570 on datacenter GPUs) |
| **AMD ROCm** | `vllm/vllm-openai-rocm` image; `--device /dev/kfd --device /dev/dri --group-add=video --cap-add=SYS_PTRACE --security-opt seccomp=unconfined` |
| **Intel XPU** | `vllm/vllm-openai-xpu` image; `--device /dev/dri --privileged` |
| **Apple Silicon** | no official image — use [vLLM-Metal](https://github.com/vllm-project/vllm-metal) (MLX backend, `mlx-community` models) |
| **CPU** | works (x86/ARM/PowerPC) via pip install; slow — for testing, not serving |
| **Google TPU / Ascend NPU** | separate packages (`vllm-tpu`, vLLM-Ascend plugin) |

VRAM math is in `02-hardware-and-vram` — the short version: weights + KV cache must fit, and vLLM's `--gpu-memory-utilization` (default 0.92) decides how much of the GPU it claims.

## pip / conda (reference only)

Not used by this repo's examples, but for completeness (native install, Linux, Python 3.11–3.14):

```bash
# NVIDIA: the official recommendation is uv
uv venv --python 3.12 --seed && source .venv/bin/activate
uv pip install vllm --torch-backend=auto   # auto-picks the torch build for your CUDA driver
vllm download-kernels                       # preinstall FlashInfer kernels (saves minutes at startup)

# one-off without a permanent env
uv run --with vllm vllm --help

# conda
conda create -n vllm python=3.12 -y && conda activate vllm
pip install --upgrade uv && uv pip install vllm --torch-backend=auto

# AMD ROCm (pip): extra index
uv pip install vllm --extra-index-url https://wheels.vllm.ai/rocm/
```

`--torch-backend=auto` inspects your CUDA driver and selects the matching PyTorch wheel (`cu126`, `cu128`, …); set `--torch-backend=cu126` to pin.

## Gotchas

1. **Entry point is `vllm serve`** — everything after the image tag is a serve/engine argument; to run arbitrary Python (offline inference), override with `--entrypoint`.
2. **`--ipc=host`** — without it (or `--shm-size`), tensor-parallel setups crash on shared memory.
3. **HF cache is ephemeral** — mount `~/.cache/huggingface:/root/.cache/huggingface` or every container re-downloads the model.
4. **Compile cache is ephemeral too** — torch.compile/Triton artifacts live in `~/.cache/vllm`; mount a volume there or each cold start recompiles (see `notes/docker.md`).
5. **`latest` ≠ stable** — pin `v0.32.0`-style tags; the Docker Hub `latest` tag follows the newest release.
6. **Gated models** — pass `--env "HF_TOKEN=$HF_TOKEN"` (or mount the token file); the server-side `--hf-token` flag also works.
7. **CUDA forward compat** — old driver + new image: set `VLLM_ENABLE_CUDA_COMPATIBILITY=1` (datacenter GPUs only).

## Sources

- [Using Docker](https://docs.vllm.ai/en/stable/deployment/docker/)
- [Quickstart](https://docs.vllm.ai/en/stable/getting_started/quickstart/)
- [GPU installation guide](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/)
- [vllm/vllm-openai tags](https://hub.docker.com/r/vllm/vllm-openai/tags)
- [vllm/vllm-openai-rocm tags](https://hub.docker.com/r/vllm/vllm-openai-rocm/tags)

## My experiments

- (fill in: e.g. "`vllm/vllm-openai:v0.32.0` + `--gpus all` on RTX 5090: Qwen3.8-27B (HF, FP8) served at X t/s; cold start Y s with mounted compile cache vs Z s without")
