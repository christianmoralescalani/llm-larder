# llama-server — deep dive

**Summary.** `llama-server` is the HTTP server: OpenAI-compatible endpoints (`/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/responses`), an Anthropic-compatible `/v1/messages`, native endpoints (`/completion`, `/embedding`, `/tokenize`, …), a built-in web UI, and Prometheus metrics. It does continuous batching across parallel *slots*, so one process serves many concurrent chats. Default bind is **127.0.0.1:9931** — always pass `--host 0.0.0.0 --port 8080` in Docker. Every flag has a `LLAMA_ARG_*` env var (CLI wins over env). Full flag tables: `cheatsheets/llama-server-flags.md`.

## Quick start (Docker)

```bash
# serve a local GGUF on the GPU, OpenAI-compatible on :8080
docker run -d --name llm -p 8080:8080 --gpus all \
  -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --host 0.0.0.0 --port 8080 \
  --n-gpu-layers -1 --ctx-size 32768 --parallel 2 --alias mymodel

# or pull straight from Hugging Face (no local file)
docker run -d --name llm -p 8080:8080 --gpus all \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  -hf ggml-org/Qwen3.5-0.8B-GGUF:Q4_K_M --host 0.0.0.0 --port 8080

curl http://localhost:8080/v1/models
curl http://localhost:8080/v1/chat/completions -d '{
  "model": "mymodel",
  "messages": [{"role": "user", "content": "hello"}],
  "stream": false
}'
```

Web UI: `http://localhost:8080/` (disable with `--no-ui`).

## Hosting & network

| Flag | Default | Notes |
|---|---|---|
| `--host` | `127.0.0.1` | comma-separated IPs or `.sock` paths; `0.0.0.0` for Docker |
| `--port` | `9931` | |
| `--api-key` | none | comma-separated list; clients send `Authorization: Bearer <key>`; also `--api-key-file` |
| `--api-prefix` | (empty) | serve under a path, e.g. `/llm` for reverse proxies |
| `--cors-origins` | `*` | restrict for the web UI; `localhost` is a special value |
| `--ssl-cert-file` / `--ssl-key-file` | none | HTTPS (needs `LLAMA_OPENSSL=ON` build) |
| `--timeout` | 3600 s | read/write timeout |
| `--threads-http` | auto | HTTP worker threads |

## Model loading

| Flag | Notes |
|---|---|
| `-m, --model FNAME` | GGUF path (local or mounted) |
| `-hf, --hf-repo user/model[:quant]` | download from Hugging Face; quant default `Q4_K_M`; mmproj auto-downloaded if present (`--no-mmproj` to skip) |
| `-hff, --hf-file FILE` | exact file inside the HF repo |
| `-hft, --hf-token` / `HF_TOKEN` | private repos |
| `--model-url` | direct URL download |
| `-dr, --docker-repo` | Docker Hub model repo |
| `--offline` | cache-only, no network |
| `--alias` | comma-separated names the API answers to (used as `model` in requests) |
| `--tags` | informational tags (not routing) |
| `--check-tensors` | validate GGUF data on load (debugging) |

## Memory & VRAM

| Flag | Default | Notes |
|---|---|---|
| `-ngl, --n-gpu-layers` | `auto` | `auto`/`all`/`-1` = all layers; exact N to split CPU/GPU |
| `--fit` | `on` | auto-adjusts unset args to fit VRAM; `--fit off` to take control; `--fit-target` margin MiB |
| `-sm, --split-mode` | `layer` | `none` (1 GPU), `layer` (pipelined), `row` (parallel), `tensor` (experimental) |
| `-ts, --tensor-split` | — | per-GPU fractions, e.g. `3,1` |
| `-mg, --main-gpu` | 0 | where KV + intermediates live |
| `-dev, --device` | all | comma list; `--list-devices` to see them |
| `-cmoe, --cpu-moe` / `-ncmoe, --n-cpu-moe N` | off | keep MoE expert weights in RAM (big VRAM saver for MoE models) |
| `-ctk` / `-ctv, --cache-type-k/v` | `f16` | KV cache dtype: `f16`, `q8_0`, `q4_0`… (q4_0 ≈ 4× smaller KV) |
| `-kvo, --kv-offload` | on | KV cache on GPU |
| `-lm, --load-mode` | auto | `mmap`, `mlock`, `mmap+mlock`, `none`, `dio` |
| `--repack` | on | weight repacking |
| `--mlock` (via load-mode) | — | keep model in RAM, no swap |

Rule of thumb for GPU: `--n-gpu-layers -1 --flash-attn on --cache-type-k q4_0 --cache-type-v q4_0` (flash-attn + q4_0 KV is the standard VRAM-saving combo; see 02-hardware-and-vram for the math).

## Context, batching, parallelism

| Flag | Default | Notes |
|---|---|---|
| `-c, --ctx-size` | model default | total context; with `--parallel N` it's the *per-slot* budget |
| `-n, --n-predict` | -1 (∞) | max tokens per request |
| `-b, --batch-size` | 2048 | logical max batch (prompt processing) |
| `-ub, --ubatch-size` | 512 | physical max batch |
| `-np, --parallel` | auto | number of slots (concurrent sequences) |
| `-cb, --cont-batching` | on | continuous/dynamic batching — the reason llama.cpp serves well |
| `-fa, --flash-attn` | auto | `on`/`off`; needs `--cache-type-k/v` ≤ f16 for the win |
| `--kv-unified` | auto | single shared KV buffer across slots (better packing) |
| `--kv-unified-per-slot N` | — | per-slot cap on the shared pool |
| `--cache-reuse N` | 0 | min chunk size to reuse from prompt cache via KV shifting |
| `--context-shift` | off | slide context for infinite generation |
| `--warmup` | on | empty warmup run at startup |
| `-t, --threads` / `-tb, --threads-batch` | auto | CPU threads (decode / prompt) |
| `--numa` | off | `distribute`/`isolate`/`numactl` for multi-socket CPUs |

## Sampling (server-wide defaults)

Per-request overrides exist in the API (`temperature`, `top_k`, `top_p`, `min_p`, `seed`, `stop`, …). Server defaults:

| Flag | Default |
|---|---|
| `--temp` | 0.80 |
| `--top-k` | 40 (0 = off) |
| `--top-p` | 0.95 |
| `--min-p` | 0.05 |
| `--repeat-penalty` | 1.00 (off) |
| `--presence-penalty` / `--frequency-penalty` | 0.00 |
| `-s, --seed` | -1 (random) |
| `--samplers` | `penalties;dry;top_n_sigma;top_k;typ_p;top_p;min_p;xtc;temperature` |
| `--grammar` / `-j, --json-schema` | — | BNF grammar / JSON-schema constrained generation |
| `--ignore-eos` | off | keep generating past EOS |

## Chat templates & reasoning

- `--jinja` (default **on**): use the model's Jinja chat template from GGUF metadata. This is what makes reasoning models, tool calling, and per-model system-prompt quirks work correctly. If a model misbehaves, first check `--jinja` is on (it is by default) and the GGUF carries the template.
- `--chat-template NAME` / `--chat-template-file FILE`: override the template (built-in names: `llama3`, `chatml`, `mistral-v3`, `deepseek3`, `kimi-k2`, …).
- `--reasoning [on|off|auto]` (default `auto` = detect from template): thinking on/off for reasoning models.
- `--reasoning-format` (`none`/`deepseek`/`deepseek-legacy`): where thoughts land in the API response (`message.reasoning_content` vs left in `content`).
- `--reasoning-effort` (`minimal`…`max`), `--reasoning-budget N` (token budget, -1 = unlimited): map onto template variables.
- `--skip-chat-parsing`: force pure content parser (everything in `content`).
- `--prefill-assistant` (default on): treat a trailing assistant message as a prefix to continue, not a full message.

## Multimodal (vision)

- `-mm, --mmproj FILE`: projector GGUF (e.g. `mmproj-BF16.gguf`). With `-hf`, mmproj is auto-downloaded if the repo has one.
- `--mmproj-offload` (default on) + `--mmproj-device`: GPU offload of the vision encoder.
- `--image-min-tokens` / `--image-max-tokens`: dynamic-resolution bounds for vision models.
- Video: `--video-fps` (default 4), `--video-ffmpeg-dir` (ffmpeg/ffprobe needed for video input).
- See `docs/multimodal.md` upstream and the mtmd tool README for the full picture.

## Speculative decoding

`--spec-type` selects the mechanism (comma-separated, default `none`):

| Type | What it is |
|---|---|
| `draft-simple` | separate small draft model (`--spec-draft-model` / `-md`, or `-hfd` for HF) |
| `draft-eagle3` | EAGLE-3 head-based drafting |
| `draft-mtp` | Multi-Token-Prediction head shipped inside some models (e.g. Qwen3.8 MTP draft) |
| `draft-dflash`, `draft-dspark` | DFlash / DSpark variants |
| `ngram-simple`, `ngram-map-k`, `ngram-map-k4v`, `ngram-mod`, `ngram-cache` | no extra model: reuse n-grams already in context (great for code/boilerplate) |

Key flags: `--spec-draft-n-max N` (draft tokens per step, default 3), `--spec-draft-n-min`, `--spec-draft-p-min` (greedy accept threshold), `--spec-draft-sampling {greedy,probabilistic}`, `--spec-draft-model`/`-hfd` (draft source), `--spec-draft-ngl` (draft VRAM offload), `--spec-draft-n-ubatch` (spare VRAM). Full deep dive: `notes/speculative-decoding.md`.

## LoRA

- `--lora FNAME[,FNAME…]`: load adapter(s) at startup (comma-separated for multiple).
- `--lora-scaled FNAME:SCALE,…`: adapters with explicit scaling.
- `--lora-init-without-apply`: load but don't apply; switch at runtime with `POST /lora-adapters` (list with `GET /lora-adapters`).

## Slots, monitoring, ops

- `GET /slots`: per-slot state (which request is running, tokens, prompt cache reuse).
- `POST /slots/{id}?action=save|restore|erase`: persist/restore a slot's KV cache to `--slot-save-path` (replaces the old `llama-save-load-state` tool).
- `--metrics` → `GET /metrics`: Prometheus (tokens/s, slot states, VRAM, …).
- `--props` → `POST /props`: change global properties at runtime.
- `--sleep-idle-seconds N`: unload/sleep after idle (VRAM reclamation).
- `--log-prompts-dir PATH`: dump prompts for debugging.
- `--media-path PATH`: allow `file://` media references from a directory.
- `--ui-config` / `--ui-config-file`: preconfigure the web UI.
- Router mode (multi-model in one server): `--models-dir`, `--models-preset` (INI), `--models-max N`, `--models-memory-margin N`, `--models-autoload`; manage with `GET /models`, `POST /models/load`, `POST /models/unload`.
- `--agent` / `--tools` / `--mcp-servers-*`: built-in agent tools & MCP (experimental; auto-restricts CORS to localhost — don't expose untrusted).

## Gotchas

1. **127.0.0.1:9931 default** — Docker `-p 8080:8080` does nothing unless you pass `--host 0.0.0.0 --port 8080`.
2. **`--ctx-size` is per-slot** when `--parallel N` is set; total KV = ctx × parallel × dtype. Size it with the VRAM math from 02-hardware-and-vram.
3. **`--n-gpu-layers auto` vs `-1`** — `auto` fits what it can; `-1` (all) is what you usually want on GPU, but watch KV cache: a huge `--ctx-size` can push KV back to CPU and tank speed.
4. **`--flash-attn on` needs quantized KV to matter** — with `f16` KV the win is small; pair with `--cache-type-k q4_0`.
5. **`--jinja` off = broken reasoning models** — thinking tags leak into `content`, tool calls don't parse. Leave it on.
6. **`--parallel` vs VRAM** — more slots = more KV; `--kv-unified` packs them better than fixed per-slot buffers.
7. **No auth by default** — `--api-key` before you expose the port beyond localhost.
8. **`--alias` mismatch** — OpenAI clients must send the `model` name matching `--alias` (or the GGUF name); wrong name → 404.
9. **Slot save/restore requires `--slot-save-path`** or the `?action=save` request fails.

## Sources

- [Server README (auto-generated flag reference)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Docker docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/docker.md)
- [Speculative decoding](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)
- [Multimodal](https://github.com/ggml-org/llama.cpp/blob/master/docs/multimodal.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B Q4_K_XL + MTP draft on :server-cuda — `--ctx-size 131072 --flash-attn on --cache-type-k q4_0 --cache-type-v q4_0 --spec-type draft-mtp --spec-draft-n-max 2`: loads in X s, prompt Y tok/s, decode Z tok/s, web UI + OpenAI client both work; `--parallel 2` cut throughput per slot by ~W%")
