# References — 05-vllm

Official docs and the tools this section is based on.

## vLLM (official)

- [Docs home (stable)](https://docs.vllm.ai/en/stable/) — entry point; "latest" tracks the dev branch
- [Quickstart](https://docs.vllm.ai/en/stable/getting_started/quickstart/) — offline + online in two examples
- [Using Docker](https://docs.vllm.ai/en/stable/deployment/docker/) — official images, tags, Podman, non-root, custom images
- [GPU installation guide](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/) — pip/uv/conda, CUDA/ROCm/XPU/TPU/NPU
- [vllm CLI guide](https://docs.vllm.ai/en/stable/cli/) — subcommands, human-readable ints, `--help=` search
- [vllm serve — full CLI reference](https://docs.vllm.ai/en/stable/cli/serve/) — every flag with defaults (Frontend, ModelConfig, CacheConfig, SchedulerConfig, ParallelConfig, …)
- [Server Arguments (YAML config)](https://docs.vllm.ai/en/stable/configuration/serve_args/) — `--config config.yaml` format
- [Optimization & tuning](https://docs.vllm.ai/en/stable/configuration/optimization/) — optimization levels, faster startup (compile cache), CUDA graphs
- [OpenAI-Compatible Server](https://docs.vllm.ai/en/stable/serving/online_serving/openai_compatible_server/) — endpoints, auth, what's supported
- [Supported models](https://docs.vllm.ai/en/stable/models/supported_models/) — auto-generated architecture matrix
- [Quantization](https://docs.vllm.ai/en/stable/features/quantization/) — AWQ/GPTQ/FP8/GGUF/bnb/LLM Compressor/…
- [Speculative decoding](https://docs.vllm.ai/en/stable/features/speculative_decoding/) — EAGLE, MTP, n-gram, suffix, draft models
- [Parallelism & scaling](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/) — TP/PP/DP/EP decision guidance
- [Benchmark CLI](https://docs.vllm.ai/en/stable/benchmarking/cli/) — `bench serve/throughput/latency/forward/sweep`, datasets, latency-metric definitions
- [Metrics design doc](https://docs.vllm.ai/en/stable/design/metrics/) — server-level vs request-level, Grafana dashboard metric list
- [Metrics setup](https://docs.vllm.ai/en/stable/examples/observability/metrics/) — Prometheus/Grafana collection
- [Prometheus & Grafana example](https://docs.vllm.ai/en/stable/examples/observability/prometheus_grafana/) — reference dashboard
- [Automatic Prefix Caching design](https://docs.vllm.ai/en/stable/design/prefix_caching/)
- [PagedAttention design](https://docs.vllm.ai/en/stable/design/paged_attention/)
- [GitHub repo](https://github.com/vllm-project/vllm) — releases, issues, `vllm/model_executor/models`
- [vLLM Recipes](https://recipes.vllm.ai/) — model+hardware → `config.yaml` + `env.sh`

## Images & registries

- [vllm/vllm-openai (Docker Hub)](https://hub.docker.com/r/vllm/vllm-openai/tags) — NVIDIA image tags (`latest`, `v<version>`, arch tags)
- [vllm/vllm-openai-rocm (Docker Hub)](https://hub.docker.com/r/vllm/vllm-openai-rocm/tags) — AMD ROCm image
- [vllm/vllm-openai-xpu (Docker Hub)](https://hub.docker.com/r/vllm/vllm-openai-xpu/tags) — Intel XPU image
- [Hugging Face: vllm-project](https://huggingface.co/vllm-project) — reference quants, EAGLE/draft models, speculators

## Related

- [PagedAttention paper (SOSP 2023)](https://arxiv.org/abs/2309.06180)
- [vLLM blog: PagedAttention post](https://blog.vllm.ai/2023/06/20/vllm.html)
- [Continuous batching (Anyscale)](https://www.anyscale.com/blog/continuous-batching-llm-inference)
- [LLM Compressor](https://github.com/vllm-project/llm-compressor) — create FP8/INT8/INT4 checkpoints
- [GuideLLM](https://github.com/guidellm/guidellm) — production server benchmarking (SLO-aware)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) — `--gpus all`
- [OpenAI API reference](https://platform.openai.com/docs/api-reference) — what `/v1/*` is compatible with
- [vLLM-Metal](https://github.com/vllm-project/vllm-metal) — Apple Silicon (MLX)

## In-repo

- `02-hardware-and-vram/` — VRAM math, quant sizes, KV cache (the numbers behind `--gpu-memory-utilization`/`--max-model-len`/`--kv-cache-dtype`)
- `01-huggingface/` — finding HF checkpoints (AWQ/FP8/GPTQ), gated repos, tokens
- `03-ollama/` — the zero-friction alternative (same models, registry experience)
- `04-llama-cpp/` — the single-user alternative (GGUF, imatrix, MTP drafts); GGUFs cross-load into vLLM with `-q gguf`
