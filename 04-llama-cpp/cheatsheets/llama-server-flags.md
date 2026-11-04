# Cheatsheet — llama-server flags

**TL;DR.** `llama-server` = OpenAI-compatible HTTP server + web UI + Prometheus. The 8 flags you actually change: `--host 0.0.0.0 --port 8080` (Docker!), `--model`/`-hf`, `--n-gpu-layers -1`, `--ctx-size`, `--parallel`, `--flash-attn on --cache-type-k q4_0 --cache-type-v q4_0`, `--jinja --reasoning on`. Every flag has a `LLAMA_ARG_*` env var (CLI wins). Verified against llama.cpp master (v0.8.0, 2026-10-28), 2026-11-03.

## Hosting & network

| Flag | Default | Notes | Env |
|---|---|---|---|
| `--host` | `127.0.0.1` | comma-separated IPs / `.sock` paths; `0.0.0.0` for Docker | `LLAMA_ARG_HOST` |
| `--port` | `9931` | | `LLAMA_ARG_PORT` |
| `--api-key` | none | comma-separated; `Authorization: Bearer <key>` | `LLAMA_API_KEY` |
| `--api-key-file` | none | one key per line, `#` comments | `LLAMA_ARG_API_KEY_FILE` |
| `--api-prefix` | (empty) | serve under a path (reverse proxy) | `LLAMA_ARG_API_PREFIX` |
| `--cors-origins` | `*` | `localhost` = special value | `LLAMA_ARG_CORS_ORIGINS` |
| `--ssl-cert-file` / `--ssl-key-file` | none | HTTPS (OpenSSL build) | `LLAMA_ARG_SSL_*` |
| `--timeout` | 3600 s | read/write | `LLAMA_ARG_TIMEOUT` |
| `--threads-http` | auto | HTTP workers | `LLAMA_ARG_THREADS_HTTP` |
| `--sse-ping-interval` | 30 s | SSE keepalive (-1 off) | `LLAMA_ARG_SSE_PING_INTERVAL` |
| `--reuse-port` | off | multiple sockets on one port | `LLAMA_ARG_REUSE_PORT` |

## Model loading

| Flag | Default | Notes | Env |
|---|---|---|---|
| `-m, --model` | — | GGUF path | `LLAMA_ARG_MODEL` |
| `-hf, --hf-repo user/model[:quant]` | — | HF download; quant default `Q4_K_M`; mmproj auto | `LLAMA_ARG_HF_REPO` |
| `-hff, --hf-file` | — | exact file in repo | `LLAMA_ARG_HF_FILE` |
| `-hft, --hf-token` | `HF_TOKEN` | private repos | `HF_TOKEN` |
| `--model-url` | — | direct URL | `LLAMA_ARG_MODEL_URL` |
| `-dr, --docker-repo` | — | Docker Hub repo | `LLAMA_ARG_DOCKER_REPO` |
| `--offline` | off | cache-only | `LLAMA_ARG_OFFLINE` |
| `--alias` | — | API model name(s), comma-separated | `LLAMA_ARG_ALIAS` |
| `--tags` | — | informational tags | `LLAMA_ARG_TAGS` |
| `--check-tensors` | off | validate GGUF data | — |
| `--override-kv` | — | `KEY=TYPE:VALUE` metadata overrides | — |

## Memory & VRAM

| Flag | Default | Notes | Env |
|---|---|---|---|
| `-ngl, --n-gpu-layers` | `auto` | `auto`/`all`/`-1`/exact N | `LLAMA_ARG_N_GPU_LAYERS` |
| `--fit` | `on` | auto-fit to VRAM; `--fit off` for manual | `LLAMA_ARG_FIT` |
| `--fit-target` | 1024 MiB | margin per device | `LLAMA_ARG_FIT_TARGET` |
| `--fit-ctx` | 4096 | min ctx `--fit` may set | `LLAMA_ARG_FIT_CTX` |
| `-sm, --split-mode` | `layer` | `none`/`layer`/`row`/`tensor`(exp) | `LLAMA_ARG_SPLIT_MODE` |
| `-ts, --tensor-split` | — | per-GPU fractions `3,1` | `LLAMA_ARG_TENSOR_SPLIT` |
| `-mg, --main-gpu` | 0 | KV + intermediates home | `LLAMA_ARG_MAIN_GPU` |
| `-dev, --device` | all | comma list; `--list-devices` | `LLAMA_ARG_DEVICE` |
| `-cmoe, --cpu-moe` | off | all MoE experts in RAM | `LLAMA_ARG_CPU_MOE` |
| `-ncmoe, --n-cpu-moe` | 0 | first N MoE layers in RAM | `LLAMA_ARG_N_CPU_MOE` |
| `--moe-cache-mib` | 0 | GPU cache for CPU-resident experts | `LLAMA_ARG_MOE_CACHE_MIB` |
| `-ctk` / `-ctv` | `f16` | KV dtypes: `f16 q8_0 q4_0 q4_1 q5_0 q5_1 iq4_nl bf16 f32` | `LLAMA_ARG_CACHE_TYPE_K/V` |
| `-kvo, --kv-offload` | on | KV on GPU | `LLAMA_ARG_KV_OFFLOAD` |
| `-lm, --load-mode` | auto | `mmap`/`mlock`/`mmap+mlock`/`none`/`dio` | `LLAMA_ARG_LOAD_MODE` |
| `--repack` | on | weight repacking | `LLAMA_ARG_REPACK` |
| `-ot, --override-tensor` | — | `pattern=buffer_type` | `LLAMA_ARG_OVERRIDE_TENSOR` |
| `--op-offload` | on | host tensor ops to device | — |

## Context, batching, parallelism

| Flag | Default | Notes | Env |
|---|---|---|---|
| `-c, --ctx-size` | model | **per-slot** with `--parallel` | `LLAMA_ARG_CTX_SIZE` |
| `-n, --n-predict` | -1 (∞) | max tokens per request | `LLAMA_ARG_N_PREDICT` |
| `-b, --batch-size` | 2048 | logical max batch | `LLAMA_ARG_BATCH` |
| `-ub, --ubatch-size` | 512 | physical max batch | `LLAMA_ARG_UBATCH` |
| `-np, --parallel` | auto | slots (concurrent seqs) | `LLAMA_ARG_N_PARALLEL` |
| `-cb, --cont-batching` | on | continuous batching | `LLAMA_ARG_CONT_BATCHING` |
| `-fa, --flash-attn` | auto | `on`/`off` — pair with q4_0 KV | `LLAMA_ARG_FLASH_ATTN` |
| `-kvu, --kv-unified` | auto | shared KV pool across slots | `LLAMA_ARG_KV_UNIFIED` |
| `--kv-unified-per-slot` | — | per-slot cap on shared pool | `LLAMA_ARG_KV_UNIFIED_PER_SLOT` |
| `--cache-reuse` | 0 | min chunk for KV-shift reuse | `LLAMA_ARG_CACHE_REUSE` |
| `--cache-prompt` | on | prompt caching | `LLAMA_ARG_CACHE_PROMPT` |
| `--context-shift` | off | slide ctx for infinite gen | `LLAMA_ARG_CONTEXT_SHIFT` |
| `--warmup` | on | empty warmup run | — |
| `--swa-full` | off | full-size SWA cache | `LLAMA_ARG_SWA_FULL` |
| `-t, --threads` | auto | decode threads | `LLAMA_ARG_THREADS` |
| `-tb, --threads-batch` | =-t | prompt threads | `LLAMA_ARG_THREADS_BATCH` |
| `--numa` | off | `distribute`/`isolate`/`numactl` | `LLAMA_ARG_NUMA` |
| `-lzm, --lazy-mode` | auto | on-demand tensor reads (mmap) | `LLAMA_ARG_LAZY_MODE` |
| `--rpc` | — | RPC servers `host:port` | `LLAMA_ARG_RPC` |

## Sampling (server defaults; per-request overrides in the API)

| Flag | Default | Notes | Env |
|---|---|---|---|
| `--temp` | 0.80 | | `LLAMA_ARG_TEMPERATURE` |
| `--top-k` | 40 | 0 = off | `LLAMA_ARG_TOP_K` |
| `--top-p` | 0.95 | 1.0 = off | `LLAMA_ARG_TOP_P` |
| `--min-p` | 0.05 | 0.0 = off | `LLAMA_ARG_MIN_P` |
| `--typical-p` | 1.00 | 1.0 = off | — |
| `--repeat-penalty` | 1.00 | 1.0 = off | `LLAMA_ARG_REPEAT_PENALTY` |
| `--presence-penalty` | 0.00 | | `LLAMA_ARG_PRESENCE_PENALTY` |
| `--frequency-penalty` | 0.00 | | `LLAMA_ARG_FREQUENCY_PENALTY` |
| `--repeat-last-n` | 64 | 0 = off | — |
| `-s, --seed` | -1 (random) | | — |
| `--samplers` | `penalties;dry;top_n_sigma;top_k;typ_p;top_p;min_p;xtc;temperature` | order matters | — |
| `--sampler-seq` | `edskypmxt` | simplified sequence | — |
| `--mirostat` | 0 | 1/2 = Mirostat 1/2 | — |
| `--mirostat-lr` / `--mirostat-ent` | 0.10 / 5.00 | eta / tau | — |
| `--dry-multiplier` / `--dry-base` | 0.00 / 1.75 | DRY sampling | — |
| `-l, --logit-bias` | — | `ID±BIAS` | — |
| `--grammar` / `--grammar-file` | — | BNF constraint | — |
| `-j, --json-schema` / `-jf` | — | JSON-schema constraint | — |
| `--ignore-eos` | off | generate past EOS | — |
| `-bs, --backend-sampling` | off | backend sampling (exp) | `LLAMA_ARG_BACKEND_SAMPLING` |

## Server-specific

| Flag | Default | Notes | Env |
|---|---|---|---|
| `--ui, --webui` | on | built-in web UI | `LLAMA_ARG_UI` |
| `--ui-config` / `--ui-config-file` | — | preconfigure UI (JSON) | `LLAMA_ARG_UI_CONFIG(_FILE)` |
| `--path` | — | static files dir | `LLAMA_ARG_STATIC_PATH` |
| `--jinja` | **on** | Jinja chat templates | `LLAMA_ARG_JINJA` |
| `--chat-template` / `--chat-template-file` | model meta | built-ins: llama3, chatml, deepseek3, kimi-k2, … | `LLAMA_ARG_CHAT_TEMPLATE(_FILE)` |
| `--chat-template-kwargs` | — | extra JSON template params | `LLAMA_ARG_CHAT_TEMPLATE_KWARGS` |
| `--reasoning` | `auto` | `on`/`off`/`auto` (detect) | `LLAMA_ARG_REASONING` |
| `--reasoning-format` | auto | `none`/`deepseek`/`deepseek-legacy` | `LLAMA_ARG_THINK` |
| `--reasoning-effort` | default | `minimal`…`max` | `LLAMA_ARG_REASONING_EFFORT` |
| `--reasoning-budget` | -1 | thinking token budget | `LLAMA_ARG_THINK_BUDGET` |
| `--reasoning-preserve` | template | keep reasoning in history | `LLAMA_ARG_REASONING_PRESERVE` |
| `--skip-chat-parsing` | off | pure content parser | `LLAMA_ARG_SKIP_CHAT_PARSING` |
| `--prefill-assistant` | on | trailing assistant msg = prefix | `LLAMA_ARG_PREFILL_ASSISTANT` |
| `-sp, --special` | off | emit special tokens | — |
| `-r, --reverse-prompt` | — | halt at string | — |
| `--embedding, --embeddings` | off | embedding-only mode | `LLAMA_ARG_EMBEDDINGS` |
| `--pooling` | model | `none`/`mean`/`cls`/`last`/`rank` | `LLAMA_ARG_POOLING` |
| `--embd-normalize` | 2 | -1 none, 0 int16, 1 taxicab, 2 euclidean, >2 p-norm | — |
| `--rerank, --reranking` | off | enable `/reranking` | `LLAMA_ARG_RERANKING` |
| `--metrics` | off | Prometheus `/metrics` | `LLAMA_ARG_ENDPOINT_METRICS` |
| `--props` | off | `POST /props` | `LLAMA_ARG_ENDPOINT_PROPS` |
| `--slots` | on | `/slots` monitoring | `LLAMA_ARG_ENDPOINT_SLOTS` |
| `--slot-save-path` | off | dir for slot KV save/restore | `LLAMA_ARG_SLOT_SAVE_PATH` |
| `--sleep-idle-seconds` | -1 (off) | sleep after idle | — |
| `--media-path` | off | `file://` media dir | — |
| `--log-prompts-dir` | off | dump prompts | — |
| `--sps, --slot-prompt-similarity` | 0.10 | slot reuse threshold | — |
| `-ctxcp, --ctx-checkpoints` | 32 | context checkpoints per slot | `LLAMA_ARG_CTX_CHECKPOINTS` |
| `-cram, --cache-ram` | 8192 MiB | max prompt-cache size (-1 unlimited) | `LLAMA_ARG_CACHE_RAM` |
| `--cache-idle-slots` | on | save idle slots to prompt cache | `LLAMA_ARG_CACHE_IDLE_SLOTS` |
| `--spm-infill` | off | SPM infill pattern | — |
| `--rope-scaling` / `--rope-scale` | model | `none`/`linear`/`yarn` | `LLAMA_ARG_ROPE_*` |
| `--yarn-*` | model | YaRN params (orig-ctx, ext-factor, attn-factor, betas) | `LLAMA_ARG_YARN_*` |

## Multimodal

| Flag | Default | Notes | Env |
|---|---|---|---|
| `-mm, --mmproj` | — | projector GGUF (auto with `-hf`) | `LLAMA_ARG_MMPROJ` |
| `-mmu, --mmproj-url` | — | projector URL | `LLAMA_ARG_MMPROJ_URL` |
| `--mmproj-auto` | on | auto-download mmproj with `-hf` (`--no-mmproj`) | `LLAMA_ARG_MMPROJ_AUTO` |
| `--mmproj-offload` | on | GPU offload encoder | `LLAMA_ARG_MMPROJ_OFFLOAD` |
| `-mmdev, --mmproj-device` | follows `-dev` | encoder device | `MTMD_BACKEND_DEVICE` |
| `--image-min-tokens` / `--image-max-tokens` | model | dynamic-resolution bounds | `LLAMA_ARG_IMAGE_*` |
| `--mtmd-batch-max-tokens` | 1024 | image tokens per encode batch | `LLAMA_ARG_MTMD_BATCH_MAX_TOKENS` |
| `--video-fps` | 4.0 | video frame rate | `LLAMA_ARG_VIDEO_FPS` |
| `--video-timestamp-interval` | 5000 ms | | `LLAMA_ARG_VIDEO_TIMESTAMP_INTERVAL` |
| `--video-ffmpeg-dir` | PATH | ffmpeg/ffprobe dir | `LLAMA_ARG_VIDEO_FFMPEG_DIR` |

## Speculative decoding

| Flag | Default | Notes | Env |
|---|---|---|---|
| `--spec-type` | `none` | `draft-simple, draft-eagle3, draft-mtp, draft-dflash, draft-dspark, ngram-simple, ngram-map-k, ngram-map-k4v, ngram-mod, ngram-cache` (mixable) | `LLAMA_ARG_SPEC_TYPE` |
| `--spec-draft-model, -md` | — | draft GGUF | `LLAMA_ARG_SPEC_DRAFT_MODEL` |
| `--spec-draft-hf, -hfd` | — | draft from HF | `LLAMA_ARG_SPEC_DRAFT_HF_REPO` |
| `--spec-draft-n-max` | 3 | draft tokens/step | `LLAMA_ARG_SPEC_DRAFT_N_MAX` |
| `--spec-draft-n-min` | 0 | | `LLAMA_ARG_SPEC_DRAFT_N_MIN` |
| `--spec-draft-p-min` | 0.00 | greedy accept threshold | `LLAMA_ARG_SPEC_DRAFT_P_MIN` |
| `--spec-draft-p-split` | 0.10 | split probability | `LLAMA_ARG_SPEC_DRAFT_P_SPLIT` |
| `--spec-draft-sampling` | `greedy` | `greedy`/`probabilistic` | `LLAMA_ARG_SPEC_DRAFT_SAMPLING` |
| `--spec-draft-ngl, -ngld` | auto | draft VRAM offload | `LLAMA_ARG_N_GPU_LAYERS_DRAFT` |
| `--spec-draft-device, -devd` | follows | draft on another GPU | — |
| `--spec-draft-n-ubatch` | =`-ub` | draft physical batch (spare VRAM) | `LLAMA_ARG_SPEC_DRAFT_N_UBATCH` |
| `--spec-draft-type-k/v, -ctkd/-ctvd` | f16 | draft KV dtypes | `LLAMA_ARG_SPEC_DRAFT_CACHE_TYPE_K/V` |
| `--spec-draft-threads, -td` | =`-t` | draft threads | `LLAMA_ARG_SPEC_DRAFT_THREADS` |
| `--spec-ngram-simple-size-n/m` | 12 / 48 | lookup n / draft m | — |
| `--spec-ngram-map-k-size-n/m` | 12 / 48 | | — |
| `--spec-ngram-map-k4v-size-n/m` | 12 / 48 | | — |
| `--spec-ngram-*-min-hits` | 1 | | — |
| `--spec-ngram-mod-n-min/n-max/n-match` | 48 / 64 / 24 | ngram-mod window | — |
| `--spec-synth-len` / `--spec-synth-rates` | — | benchmarking only | `LLAMA_ARG_SPEC_SYNTH_*` |

## LoRA & control vectors

| Flag | Notes |
|---|---|
| `--lora FNAME[,FNAME…]` | adapter(s) at load |
| `--lora-scaled FNAME:SCALE,…` | adapters with scales |
| `--lora-init-without-apply` | load unapplied; switch via `POST /lora-adapters` |
| `--control-vector FNAME[,…]` / `--control-vector-scaled` | control vectors (optional scale, layer range) |

## Ops / router / agent

| Flag | Default | Notes | Env |
|---|---|---|---|
| `--models-dir` | off | router: model dir | `LLAMA_ARG_MODELS_DIR` |
| `--models-preset` | — | router: INI presets | `LLAMA_ARG_MODELS_PRESET` |
| `--models-max` | 4 | router: max loaded (0 = ∞) | `LLAMA_ARG_MODELS_MAX` |
| `--models-memory-margin` | 0 | router: MiB free per device, LRU unload | `LLAMA_ARG_MODELS_MEMORY_MARGIN` |
| `--models-autoload` | on | router: auto-load | `LLAMA_ARG_MODELS_AUTOLOAD` |
| `--agent` | off | CORS proxy + all built-in tools (exp; locks CORS to localhost) | `LLAMA_ARG_AGENT` |
| `--tools` | none | `all` or list: read_file, write_file, edit_file, exec_shell_command, grep_search, file_glob_search, get_info | `LLAMA_ARG_TOOLS` |
| `--tools-runtime` | none | `docker:<img>`, `ssh:<target>`, … | `LLAMA_ARG_TOOLS_RUNTIME` |
| `--mcp-servers-config` / `--mcp-servers-json` | none | MCP server defs (Cursor format) | `LLAMA_ARG_MCP_SERVERS_*` |
| `--connect` / `--connect-code` | off | P2P tunnel via llama-connect | `LLAMA_ARG_CONNECT(_CODE)` |
| `--log-file` / `--log-jsonl` / `--log-verbosity` | — / off / 3 | logging | `LLAMA_ARG_LOG_*` |
| `--version` / `--list-devices` / `-cl, --cache-list` | — | diagnostics | — |

## Env var mapping

`--flag-name` → `LLAMA_ARG_FLAG_NAME` (dashes → underscores, uppercased). Exceptions: `--api-key` → `LLAMA_API_KEY`, `--hf-token` → `HF_TOKEN`, `--mmproj-device` → `MTMD_BACKEND_DEVICE`. Booleans accept `1/0, on/off, true/false, enabled/disabled`. CLI args override env vars.

## Canonical Docker run

```bash
docker run -d --name llm -p 8080:8080 --gpus all \
  -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf \
  --host 0.0.0.0 --port 8080 \
  --n-gpu-layers -1 --ctx-size 32768 --parallel 2 \
  --flash-attn on --cache-type-k q4_0 --cache-type-v q4_0 \
  --jinja --reasoning on --alias mymodel --metrics
```
