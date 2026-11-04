# Cheatsheet — llama.cpp commands

**TL;DR.** The tools are separate binaries (or subcommands of the unified `llama` binary: `llama cli`, `llama serve`). In Docker, pick the image by job: `server*` to serve, `light*` for chat, `full*` for quantize/convert/bench. Day-to-day: `llama-server` (serve), `llama-cli` (chat), `convert_hf_to_gguf.py` (HF→GGUF), `llama-quantize` (shrink), `llama-bench` (speed), `llama-perplexity` (quality), `llama-imatrix` (quant quality lever). Verified against llama.cpp master (v0.8.0, 2026-10-28), 2026-11-03.

## Tool table

| Tool | Job | Image |
|---|---|---|
| `llama-server` (`llama serve`) | HTTP server: OpenAI/Anthropic APIs, web UI, metrics | `server*` |
| `llama-cli` (`llama cli`) | terminal chat REPL, one-shot prompts, template debugging | `light*`, `full*` |
| `llama-completion` | non-interactive single completion (batch/pipe) | `full*` |
| `convert_hf_to_gguf.py` | HF weights → GGUF (also `--mmproj` for vision encoders) | `full` |
| `llama-quantize` | GGUF → target quant (imatrix-aware) | `full` |
| `llama-imatrix` | compute importance matrix from a text corpus | `full` |
| `llama-bench` | pp/tg/pg benchmarks (quants, flags, threads, ngl) | `full` |
| `llama-batched-bench` | batched throughput benchmarks | `full` |
| `llama-perplexity` | perplexity of a model on a text file | `full` |
| `llama-export-lora` | merge LoRA adapter(s) into a new GGUF | `full` |
| `llama-gguf-split` | split/merge sharded GGUF files | `full` |
| `llama-cvector-generator` | generate control vectors | `full` |
| `llama-fit-params` | fit params to device memory | `full` |
| `llama-results` | manage benchmark result storage | `full` |
| `llama-rpc` | RPC server (offload to another machine) | `full` |
| `llama-tokenize` | text → tokens (batch) | `full` |
| `llama-tts` | text-to-speech (TTS models) | `full` |
| `llama-connect` | P2P tunnel for `--connect` servers | `full` |
| `llama-mtmd-cli` | multimodal CLI (vision/audio/video) | `full` |
| `ggml-metal-tuning` | Metal kernel tuning (Apple) | `full` |

Removed since older docs: `llama-embeddings` → `llama-server --embedding` + `/v1/embeddings`; `llama-save-load-state` → `POST /slots/{id}?action=save|restore`; Makefile build (`make llamacpp`) → CMake only.

## Docker wrappers

```bash
# serve (detached)
docker run -d --name llm -p 8080:8080 --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --host 0.0.0.0 --port 8080 --n-gpu-layers -1

# chat (interactive)
docker run --rm -it --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  --model /models/model.gguf --n-gpu-layers -1

# one-shot
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF -p "hello" --single-turn

# HF → GGUF → quantize pipeline
docker run --rm -v ~/Downloads/models:/m ghcr.io/ggml-org/llama.cpp:full \
  python convert_hf_to_gguf.py --remote <org>/<model> --outfile /m/model-bf16.gguf --outtype bf16
docker run --rm -v ~/Downloads/models:/m ghcr.io/ggml-org/llama.cpp:full \
  llama-quantize /m/model-bf16.gguf /m/model-Q4_K_M.gguf Q4_K_M

# benchmark + perplexity
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda llama-bench -m /m/model-Q4_K_M.gguf
docker run --rm -v ~/Downloads/models:/m -v ~/calibration:/cal \
  ghcr.io/ggml-org/llama.cpp:full llama-perplexity -m /m/model-Q4_K_M.gguf -f /cal/wiki.txt
```

## API quick reference (server on :8080)

```bash
B=http://localhost:8080

curl -s $B/health                                  # {"status":"ok"} | 503 loading
curl -s $B/v1/models                               # model info + meta
curl -s $B/v1/chat/completions -d '{
  "model":"mymodel","messages":[{"role":"user","content":"hi"}],"stream":false}'
curl -N  $B/v1/chat/completions -d '{
  "model":"mymodel","messages":[{"role":"user","content":"count to 5"}]}'   # SSE stream
curl -s $B/v1/embeddings -d '{"model":"mymodel","input":"hello"}'
curl -s $B/tokenize -d '{"content":"hello","with_pieces":true}'
curl -s $B/apply-template -d '{"messages":[{"role":"user","content":"hi"}]}'
curl -s $B/slots                                   # per-slot state
curl -s $B/metrics | grep llama                    # prometheus (--metrics)
curl -s -X POST $B/slots/0?action=erase            # clear a slot's KV
```

OpenAI SDK: `openai.OpenAI(base_url="http://localhost:8080/v1", api_key="sk-…")` — `model` must match `--alias`.

## Speculative decoding quick start

```bash
# MTP (heads ship with the model)
--spec-type draft-mtp --spec-draft-model /models/mtp-draft.gguf --spec-draft-n-max 2

# small draft model
--spec-type draft-simple --spec-draft-model /models/draft-0.5B.gguf --spec-draft-n-max 4

# no extra model (code/boilerplate workloads)
--spec-type ngram-simple
```

## Gotchas

1. **`--host 0.0.0.0 --port 8080`** — the #1 Docker mistake (default bind 127.0.0.1:9931).
2. **`-m` vs `-hf`** — local path vs HF repo; `-hf` needs network (or a warm cache + `--offline`).
3. **`--ctx-size` is per-slot** when `--parallel N` is set.
4. **`--jinja` is on by default** — don't disable it for reasoning/tool-calling models.
5. **`llama-bench` `-ngl` defaults to -1** (all GPU) — unlike the server's `auto`.
6. **Quantize from 16-bit only** — never requantize an existing quant.
7. **`--alias` mismatch → 404** in `/v1/*` even though the model is loaded.
