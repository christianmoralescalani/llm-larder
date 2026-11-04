# Installing llama.cpp — Docker-first

**Summary.** llama.cpp is a pure C/C++ LLM/VLM inference engine (built on the [ggml](https://github.com/ggml-org/ggml) tensor library, MIT license, no runtime dependencies). In this repo the install is the official Docker image `ghcr.io/ggml-org/llama.cpp` — no native install needed, no CMake, no CUDA toolkit on the host. Prebuilt binaries and a CMake source build exist for reference. One gotcha: the legacy Makefile build was **removed** — `make` now just prints an error pointing you to CMake, so any old tutorial saying `make llamacpp` is outdated.

## What it is

- LLM **and** VLM inference (image, audio, video input via `libmtmd`).
- 1.5-, 2-, 3-, 4-, 5-, 6- and 8-bit integer quantization (IQ*/Q* formats).
- Backends: CPU (AVX/AVX2/AVX512/AMX, NEON), CUDA (NVIDIA), HIP (AMD), MUSA (Moore Threads), Metal (Apple), Vulkan, SYCL (Intel), OpenVINO, CANN (Ascend NPU), RPC, WebGPU, ZenDNN (AMD CPU).
- CPU+GPU hybrid inference for models larger than total VRAM.
- Ships a unified `llama` binary with subcommands (`llama cli`, `llama serve`) plus the individual tools (`llama-cli`, `llama-server`, `llama-quantize`, …) — see `notes/tools-inventory.md`.

## Option 1 — Docker (the install for this repo)

Three base images, each in CPU and GPU variants (full list in `notes/docker.md`):

| Image | Contents | Use for |
|---|---|---|
| `ghcr.io/ggml-org/llama.cpp:server` | only `llama-server` | serving (OpenAI-compatible API + web UI) |
| `ghcr.io/ggml-org/llama.cpp:light` | `llama-cli` + `llama-completion` | interactive chat / one-shot |
| `ghcr.io/ggml-org/llama.cpp:full` | everything incl. Python conversion tools | quantizing, converting, benchmarking |

GPU variants: `:server-cuda` / `:server-cuda13` (NVIDIA), `:server-rocm` (AMD), `:server-vulkan` (AMD/Intel), `:server-intel` (SYCL), `:server-musa`, `:server-openvino`.

Quick start (serving a local GGUF on an NVIDIA GPU):

```bash
docker run -p 8080:8080 --gpus all \
  -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --host 0.0.0.0 --port 8080
```

Web UI at `http://localhost:8080`. Full examples in `cheatsheets/llama-cpp-docker.md`.

## Option 2 — Prebuilt binaries (reference)

- [Releases page](https://github.com/ggml-org/llama.cpp/releases) — per-OS archives (Linux x64/ARM64, macOS, Windows), stable `vX.Y.Z` and nightly `bYYYYMMDD` tags.
- Install script: `curl -LsSf https://llama.app/install.sh | sh` (or PowerShell: `irm https://llama.app/install.ps1 | iex`), or browse https://llama.app.
- Unpack and put `llama` (or the individual binaries) on your PATH:

```sh
llama cli -hf ggml-org/Qwen3.5-0.8B-GGUF   # chat, auto-downloads from HF
llama serve -hf ggml-org/Qwen3.5-0.8B-GGUF # OpenAI-compatible server
```

## Option 3 — Build from source with CMake (reference)

The Makefile build is gone (it errors out: "Build system changed: The Makefile build has been replaced by CMake"). CMake only:

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j 8
# binaries in ./build/bin/
```

Backend switches (all in [docs/build.md](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)):

| Backend | CMake flag | Notes |
|---|---|---|
| CPU (default) | — | auto-detects AVX/NEON; `-DGGML_NATIVE=ON` |
| CUDA (NVIDIA) | `-DGGML_CUDA=ON` | needs CUDA toolkit; `-DCMAKE_CUDA_ARCHITECTURES="89"` to target one arch |
| Vulkan (AMD/Intel) | `-DGGML_VULKAN=ON` | works on most GPUs |
| HIP (AMD) | `-DGGML_HIP=ON` | needs ROCm; `-DGPU_TARGETS=gfx1030` |
| SYCL (Intel) | `-DLLAMA_SYCL=ON` | see `docs/backend/SYCL.md` |
| MUSA (Moore Threads) | `-DGGML_MUSA=ON` | |
| BLAS | `-DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS` | CPU acceleration |
| RPC | `-DLLAMA_RPC=ON` | distributed CPU/GPU (proof of concept) |
| OpenSSL (server HTTPS) | `-DLLAMA_OPENSSL=ON` | for `--ssl-key-file`/`--ssl-cert-file` |

## Versioning

- Latest stable: **v0.8.0** (2026-10-28). Nightly builds publish under `b*` tags.
- Docker image tags track the release (untagged `:server` etc. = latest). Pin a version tag for reproducibility.
- Breaking changes happen between versions (e.g. `--draft` → `--spec-type` rename, `llama-embeddings` removal) — check the release notes when bumping.

## Sources

- [llama.cpp README](https://github.com/ggml-org/llama.cpp) — quick start, backends
- [Docker docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/docker.md)
- [Build docs (CMake)](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)
- [Releases](https://github.com/ggml-org/llama.cpp/releases)

## My experiments

- (fill in: e.g. "pulled `server-cuda`, ran a 7B Q4_K_M in ~30 s to first token; `llama --version` inside container says v0.8.0")
