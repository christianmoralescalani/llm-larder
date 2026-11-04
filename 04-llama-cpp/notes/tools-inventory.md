# Tool inventory — what each llama.cpp binary does

**Summary.** `llama.cpp` ships ~20 small single-purpose binaries under `tools/` plus Python conversion scripts at the repo root. In Docker they live at `/app/llama-*` (the `full` image has all of them, `light` has `llama-cli`/`llama-completion`, `server` has only `llama-server`). Since v0.8.0 there is also a unified `llama` binary with subcommands (`llama cli`, `llama serve`, …) — the individual binaries still exist and are what the Docker entrypoints use. Two tools from older tutorials are **gone**: `llama-embeddings` (embeddings moved into `llama-server --embedding` + `/v1/embeddings`) and `llama-save-load-state` (slot KV save/restore moved to the server's `/slots/{id}?action=save|restore` endpoints).

## The tools

| Binary | What it does | When you use it |
|---|---|---|
| `llama-cli` | interactive chat REPL + one-shot generation; multimodal; grammars; reasoning models | day-to-day chat, testing a model |
| `llama-completion` | non-interactive text completion (raw prompt, no chat template) | scripting, benchmarking raw completion |
| `llama-server` | HTTP server: OpenAI- + Anthropic-compatible API, web UI, parallel slots, continuous batching | serving models to apps |
| `llama-quantize` | GGUF → lower-precision GGUF (Q4_K_M, IQ2_XXS, …) | shrinking models |
| `llama-imatrix` | computes an importance matrix from calibration text, used by `llama-quantize` | high-quality quants |
| `llama-bench` | performance benchmark: prompt processing (pp) + text generation (tg), many output formats | comparing quants/flags on your hardware |
| `llama-perplexity` | perplexity / KL-divergence over a corpus (Wikitext-2 convention) | measuring quant quality loss |
| `llama-export-lora` | applies LoRA adapter(s) to a base model and exports the merged model | baking an adapter in |
| `llama-mtmd-cli` | multimodal test/dev CLI (image, audio, video) | testing vision/audio models |
| `llama-rpc-server` | RPC backend (distributed CPU/GPU inference) | advanced; proof-of-concept stage, insecure |
| `llama-tts` | text-to-speech demo via `libmtmd` | TTS models |
| `llama-tokenize` | tokenize/detokenize text | debugging templates |
| `llama-tuning` | `ggml-metal-tuning` — offline Metal kernel tuner | Apple Silicon devs |
| `llama-fit-params` | helper for the `--fit` auto-sizing feature | advanced memory fitting |
| `llama-gguf-split` | split/merge GGUF files | sharding big files |
| `llama-cvector-generator` | generates control vectors from a GGUF model | control vectors (`--control-vector`) |
| `llama-batched-bench` | batched-decoding benchmark (shared vs unshared prompt) | advanced benchmarking |
| `llama-results` | `--check` model outputs vs a previous commit | regression checking |
| `llama-connect` | peer-to-peer tunnel for a server (`--connect`) | remote access |
| `llama-ui` | modern SvelteKit web UI alternative for `llama-server` | nicer frontend |

## Python scripts (repo root)

| Script | What it does |
|---|---|
| `convert_hf_to_gguf.py` | Hugging Face (safetensors) → GGUF; the entry point of the quantization pipeline; also converts multimodal projectors (`--mmproj`) and EAGLE-3/DFlash/DSpark draft models (`--target-model-dir`) |
| `convert_lora_to_gguf.py` | HF LoRA adapter → GGUF adapter (for `--lora`) |
| `convert_llama_ggml_to_gguf.py` | legacy ggml → GGUF conversion |

In Docker, the `full` image ships these scripts plus the Python dependencies — run `python convert_hf_to_gguf.py …` directly in the container (see `notes/quantization.md`). No-setup alternative: the [GGUF-my-repo HF space](https://huggingface.co/spaces/ggml-org/gguf-my-repo).

## Removed / renamed (older tutorials)

| Old | Now |
|---|---|
| `llama-embeddings` | `llama-server --embedding` + `POST /v1/embeddings` (model needs a non-`none` pooling) |
| `llama-save-load-state` | `POST /slots/{id}?action=save` / `?action=restore` (needs `--slot-save-path`) |
| `llama-bench` (old path `examples/bench`) | `tools/llama-bench` |
| `--draft` / `--draft-n-max` | `--spec-type` + `--spec-draft-n-max` (speculative decoding rework) |
| Makefile build (`make llamacpp`) | CMake only — the Makefile now errors out and points to `docs/build.md` |

## Sources

- [tools/ directory](https://github.com/ggml-org/llama.cpp/tree/master/tools)
- [Server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) · [CLI README](https://github.com/ggml-org/llama.cpp/blob/master/tools/cli/README.md)
- [Quantize README](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md) · [Imatrix README](https://github.com/ggml-org/llama.cpp/blob/master/tools/imatrix/README.md)
- [llama-bench README](https://github.com/ggml-org/llama.cpp/blob/master/tools/llama-bench/README.md) · [Perplexity README](https://github.com/ggml-org/llama.cpp/blob/master/tools/perplexity/README.md)

## My experiments

- (fill in: e.g. "`full` image: `llama-quantize --imatrix` on my 27B took ~X min on the GPU; `llama-bench -o csv` output piped into a spreadsheet")
