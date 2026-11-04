# References — 04-llama-cpp

Official docs and the tools this section is based on.

## llama.cpp (official, GitHub)

- [README](https://github.com/ggml-org/llama.cpp/blob/master/README.md) — overview, quickstart, unified `llama` binary
- [Build docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md) — CMake, backends (CUDA/ROCm/Vulkan/HIP/SYCL/MUSA/OpenVINO/BLAS), installers
- [Docker docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/docker.md) — official images, GPU variants, local builds
- [Server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) — full auto-generated flag reference + API endpoint docs
- [CLI README](https://github.com/ggml-org/llama.cpp/blob/master/tools/cli/README.md) — CLI flag reference
- [Speculative decoding](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md) — draft/EAGLE-3/DFlash/DSpark/MTP/ngram implementations
- [Multimodal](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md) — mmproj, vision/audio/video
- [Function calling](https://github.com/ggml-org/llama.cpp/blob/master/docs/function-calling.md) — tool-call styles
- [quantize README](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md) — pipeline, options, examples
- [imatrix README](https://github.com/ggml-org/llama.cpp/blob/master/tools/imatrix/README.md)
- [llama-bench README](https://github.com/ggml-org/llama.cpp/blob/master/tools/llama-bench/README.md)
- [perplexity README](https://github.com/ggml-org/llama.cpp/blob/master/tools/perplexity/README.md)
- [export-lora README](https://github.com/ggml-org/llama.cpp/blob/master/tools/export-lora/README.md)
- [SPEED-Bench client](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/bench/speed-bench/README.md) — server-level spec-decoding benchmark
- [Wiki: templates supported by apply-template](https://github.com/ggml-org/llama.cpp/wiki/Templates-supported-by-llama_chat_apply_template)

## Images & registries

- [ghcr.io/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp/pkgs/container/llama.cpp) — official image tags
- [Hugging Face: ggml-org](https://huggingface.co/ggml-org) — reference GGUF quants (Qwen3.5, Llama, Gemma, …)
- [GGUF-my-repo Space](https://huggingface.co/spaces/ggml-org/gguf-my-repo) — build custom quants in the browser (syncs from main every 6 h)

## Related

- [ggml](https://github.com/ggml-org/ggml) — tensor library underneath
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) — `--gpus all` for CUDA images
- [OpenAI API reference](https://platform.openai.com/docs/api-reference) — what `/v1/*` is compatible with
- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages) — what `/v1/messages` mirrors
- [Speculative decoding (Wikipedia)](https://en.wikipedia.org/wiki/Transformer_(deep_learning)#Speculative_decoding)

## In-repo

- `02-hardware-and-vram/` — VRAM math, quant sizes, KV cache (the numbers behind `--ctx-size`/`--cache-type-k` decisions)
- `01-huggingface/` — finding GGUFs, MTP drafts, mmproj files
- `03-ollama/` — the managed alternative (same models, less control)
