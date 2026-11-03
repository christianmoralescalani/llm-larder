# Docs & references — 02-hardware-and-vram

Official docs and curated external references for hardware, VRAM math, and inference performance.

## Official hardware specs

- [NVIDIA GeForce RTX 50 series](https://www.nvidia.com/en-us/geforce/graphics-cards/50-series/) — VRAM/bandwidth per card (5090 32 GB/1792 GB/s … 5050 8 GB/320 GB/s)
- [NVIDIA GeForce RTX 40 series](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/)
- [NVIDIA GeForce RTX 30 series](https://www.nvidia.com/en-us/geforce/graphics-cards/30-series/)
- [NVIDIA GPU comparison tool](https://www.nvidia.com/en-us/geforce/graphics-cards/compare/)
- [Apple — MacBook Pro (M4 Pro / M4 Max) tech specs](https://support.apple.com/en-us/121553) — note the binned bandwidth: M4 Max 36 GB = 410 GB/s, 48–128 GB = 546 GB/s
- [Apple — Mac Studio (M5 Max / M5 Ultra) tech specs](https://support.apple.com/en-us/128107) — M5 Max 614 GB/s, M5 Ultra 1.2 TB/s
- [TechPowerUp GPU database](https://www.techpowerup.com/gpu-database/) — cross-check bandwidth/TDP/bus width

## Runtime docs (memory/offload sections)

- [llama.cpp — build & GPU offload](https://github.com/ggml-org/llama.cpp) — `--n-gpu-layers`, `--main-gpu`, `--tensor-split`, `--mlock`, `--cache-type-k/v`, `--flash-attn`
- [llama.cpp — quantization](https://github.com/ggml-org/llama.cpp/blob/master/docs/quantization.md) — imatrix, K-quants
- [vLLM — parallelism & KV cache](https://docs.vllm.ai/) — `--tensor-parallel-size`, `--pipeline-parallel-size`, `--kv-cache-dtype`, `--gpu-memory-utilization`
- [Ollama — environment variables](https://github.com/ollama/ollama/blob/main/docs/faq.md) — `OLLAMA_NUM_GPU`, `OLLAMA_KV_CACHE`, `CUDA_VISIBLE_DEVICES`

## Benchmark databases & calculators

- [llama.cpp discussion #4167 — Apple Silicon benchmarks](https://github.com/ggml-org/llama.cpp/discussions/4167) — the canonical M-series tok/s collection (pp/tg by chip)
- [Local LLM benchmark database](https://www.kunalganglani.com/llm-benchmarks) — 82+ measured tok/s across Mac/NVIDIA/AMD, JSON API
- [CanItRun](https://canitrun.dev/) — per-GPU "which models fit" with estimated tok/s
- [VRAM Calculator](https://vram-calc.com/) — decode tok/s estimator from memory bandwidth (roofline)
- [LLM VRAM & Performance Calculator (ApX ML)](https://apxml.com/posts/how-to-calculate-vram-requirements-for-an-llm) — the method behind the "how much VRAM" videos

## Sizing guides (good reads)

- [Modal — How much VRAM do you need for LLM inference?](https://modal.com/blog/how-much-vram-need-inference) — the `M = P × (Q/8) × 1.2` formula
- [Runpod — GPU memory sizing guide](https://www.runpod.io/articles/guides/gpu-memory-sizing-guide-for-llm-inference) — weights + KV + overhead, serving perspective
- [GPUSmith — LLM inference hardware sizing](https://gpusmith.com/articles/en/llm-inference-hardware-sizing-guide) — roofline model, bandwidth vs compute

## Curated for this topic

- [00-foundations/cheatsheets/kv-cache-formula.md](../../00-foundations/cheatsheets/kv-cache-formula.md) — the KV formula with per-model per-token table
- [00-foundations/notes/inference-pipeline.md](../../00-foundations/notes/inference-pipeline.md) — prefill vs decode, why decode is bandwidth-bound
- [01-huggingface/notes/formats-and-quantization.md](../../01-huggingface/notes/formats-and-quantization.md) — which quant providers to download from
