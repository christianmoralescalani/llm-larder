# References — 01-huggingface

Official docs and the repos/providers this section is based on.

## Hugging Face

- [hf CLI guide](https://huggingface.co/docs/huggingface_hub/guides/cli) — `hf auth/download/models/cache/repos/upload/cp/env`
- [CLI reference](https://huggingface.co/docs/huggingface_hub/package_reference/cli)
- [Hub API endpoints](https://huggingface.co/docs/hub/api) · [OpenAPI spec](https://huggingface.co/.well-known/openapi.json) (`.md` for agents)
- [Environment variables](https://huggingface.co/docs/huggingface_hub/package_reference/environment_variables) — `HF_TOKEN`, `HF_HOME`, `HF_HUB_OFFLINE`, …
- [Security tokens](https://huggingface.co/docs/hub/security-tokens)
- [Model card guide](https://huggingface.co/docs/hub/model-cards) · [Licenses list](https://huggingface.co/docs/hub/repositories-licenses)
- [huggingface_hub Python](https://huggingface.co/docs/huggingface_hub/package_reference)

## Quant providers & collections

- [ggml-org (official llama.cpp GGUFs)](https://huggingface.co/ggml-org) · [multimodal GGUFs collection](https://huggingface.co/collections/ggml-org/multimodal-ggufs-68244e01ff1f39e5bebeeedc)
- [bartowski](https://huggingface.co/bartowski) · [unsloth](https://huggingface.co/unsloth) · [mradermacher](https://huggingface.co/mradermacher) · [lmstudio-community](https://huggingface.co/lmstudio-community)
- Official orgs with GGUF repos (often gated): [Qwen](https://huggingface.co/Qwen) · [meta-llama](https://huggingface.co/meta-llama) · [deepseek-ai](https://huggingface.co/deepseek-ai) · [moonshotai](https://huggingface.co/moonshotai)

## llama.cpp (GGUF / mmproj / MTP)

- [Multimodal (mtmd) README](https://github.com/ggml-org/llama.cpp/blob/master/tools/mtmd/README.md) — what mmproj is, how to make one
- [Multimodal model list](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)
- [Speculative decoding docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md) — `--spec-type`, `draft-mtp`
- [llama-server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) — `-hf`, `--mmproj`, `--fit` flags
- [gguf-py](https://github.com/ggml-org/llama.cpp/tree/master/gguf-py) (PyPI: `gguf`)

## vLLM (MTP)

- [vLLM MTP (Multi-Token Prediction)](https://docs.vllm.ai/en/latest/features/speculative_decoding/mtp/) — `--speculative-config '{"method":"mtp",…}'`, Gemma 4 assistant checkpoints

## Papers

- [DeepSeek-V3 Technical Report](https://arxiv.org/abs/2412.19437) — MTP design
- [DeepSeek-R1](https://arxiv.org/abs/2501.12948)
- [Speculative decoding](https://arxiv.org/abs/2211.17192)

## Misc

- [hf-mirror.com](https://hf-mirror.com) — mirror for blocked regions
- [aria2](https://aria2.github.io/) — parallel/resumable direct downloads
- [transformers](https://huggingface.co/docs/transformers) — `pipeline()` quick tests
