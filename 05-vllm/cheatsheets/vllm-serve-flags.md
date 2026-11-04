# Cheatsheet — vllm serve flags

**TL;DR.** `vllm serve <model>` = OpenAI-compatible server. The ~12 flags you actually touch: `--model` (+ `--served-model-name`), `--host 0.0.0.0 --port 8000`, `--api-key`, `--max-model-len`, `--gpu-memory-utilization`, `--kv-cache-dtype`, `--max-num-seqs`, `--quantization`, `--dtype`, `--enable-prefix-caching` (on by default), `--speculative-config`, `--tensor-parallel-size`. Everything after the image tag in Docker is a serve flag. Full list: `vllm serve --help=all` (search: `--help=max`). Verified against vLLM v0.32.0 (stable, 2026-10-18), docs.vllm.ai, 2026-11-03.

## Model & identity

| Flag | Default | Notes |
|---|---|---|
| `--model` | `Qwen/Qwen3-0.6B` | HF repo ID **or local path** (safetensors or GGUF) |
| `--served-model-name` | = `--model` | name(s) clients use; comma-list ok |
| `--tokenizer` | = model | separate tokenizer |
| `--trust-remote-code` | off | run custom HF repo code |
| `--hf-token` | — | gated repos (`HF_TOKEN` env works too) |
| `--revision` | — | branch/tag/commit |
| `--load-format` | `auto` | `safetensors`, `dummy`, `tensorizer`, `runai_streamer`, `sharded_state`, `ipc_cache` |
| `--generation-config` | `model` | `vllm` = ignore repo's `generation_config.json` |
| `--override-generation-config` | — | JSON overrides, e.g. `'{"temperature":0.5}'` |
| `--max-model-len` | model max / fit | `32k`/`32K`/`25.6k` suffixes; `-1` = largest that fits |
| `--dtype` | `auto` | `bfloat16`, `float16`, `float32` |
| `--quantization`, `-q` | auto-detect | `awq`, `gptq`, `fp8`, `bitsandbytes`, `gguf`, `compressed-tensors`, `modelopt`, `torchao`, … |
| `--enforce-eager` | off | no torch.compile/CUDA graphs (debug; slower) |
| `--attention-backend` | auto | `FLASH_ATTN`, `FLASHINFER`, … |
| `--max-logprobs` | 20 | max logprobs per token |
| `--disable-sliding-window` | off | force full-attention for sliding-window models |

## Network & auth

| Flag | Default | Notes |
|---|---|---|
| `--host` | `0.0.0.0` | |
| `--port` | `8000` | |
| `--uds` | — | Unix domain socket (host/port ignored) |
| `--api-key` | none | comma-separated (rotation); guards **only** `/v1`, `/v2`, `/inference` |
| `--ssl-certfile` / `--ssl-keyfile` / `--ssl-ca-certs` | — | HTTPS / mTLS |
| `--allowed-origins` | `[]` | CORS; browser access off by default |
| `--root-path` | — | serve under a path (reverse proxy) |
| `--disable-fastapi-docs` | off | hide `/docs` |
| `--sse-keep-alive-interval` | — | SSE keep-alive seconds |

## Memory & KV cache

| Flag | Default | Notes |
|---|---|---|
| `--gpu-memory-utilization` | `0.92` | VRAM ceiling for the instance (`--device-memory-utilization` alias); per-instance — split manually for co-located instances |
| `--kv-cache-memory-bytes` | auto | explicit KV size (overrides util when set) |
| `--kv-cache-dtype` | `auto` | `fp8`, `fp8_e5m2`, `int8_per_token_head`, `nvfp4` — cut KV VRAM |
| `--block-size` | 16 | KV block tokens |
| `--enable-prefix-caching` | **on** (V1) | `--disable-prefix-caching` to turn off |
| `--mamba-cache-mode` | — | Mamba/hybrid state cache |

## Scheduling & throughput

| Flag | Default | Notes |
|---|---|---|
| `--max-num-seqs` | engine-derived (typically 256) | max concurrent sequences — main throughput dial |
| `--max-num-batched-tokens` | 8192 (derived) | max tokens per iteration (`1k`/`2M` suffixes) |
| `--enable-chunked-prefill` | **on** (V1) | slice long prefills across iterations |
| `--scheduling-policy` | `fcfs` | `priority` = lower number first |
| `--watermark` | `0.0` | KV blocks kept free to avoid preemption thrash |
| `--num-scheduler-steps` | 1 | schedule N steps ahead (latency) |
| `--enable-deterministic` | off | deterministic sampling (slower) |
| `--enable-sleep-mode` | off | free GPU memory between bursts (RL workloads) |

## Speculative decoding

| Flag | Notes |
|---|---|
| `--speculative-config`, `-sc` | JSON: `'{"model":"<draft>","method":"eagle3","num_speculative_tokens":3}'` |
| `--spec-method` | `eagle`, `eagle3`, `mtp`, `dflash`, `dspark`, `ngram`, `ngram_gpu`, `suffix`, `draft_model`, … |
| `--spec-model` | draft/EAGLE/MTP weights |
| `--spec-tokens` | draft tokens per step |

## Parallelism

| Flag | Default | Notes |
|---|---|---|
| `--tensor-parallel-size`, `-tp` | 1 | split layers across N GPUs (needs `--ipc=host` in Docker) |
| `--pipeline-parallel-size`, `-pp` | 1 | layer stages (multi-node) |
| `--data-parallel-size`, `-dp` | 1 | N replicas behind one endpoint |
| `--enable-expert-parallel`, `-ep` | off | MoE expert sharding |
| `--enable-eplb` | off | dynamic expert load balancing |
| `--nnodes`, `-n` | 1 | multi-node |
| `--node-rank`, `-r` | 0 | this node's rank |
| `--master-addr` / `--master-port` | — | multi-node rendezvous |
| `--device-ids` | — | specific GPU ids |
| `--numa-bind` | off | NUMA pinning (CPU) |

## Chat, tools & multimodal

| Flag | Notes |
|---|---|
| `--chat-template` | override tokenizer template (file/Jinja) |
| `--default-chat-template-kwargs` | e.g. `'{"enable_thinking": false}'` (Qwen3/DeepSeek) |
| `--response-role` | override response role name |
| `--enable-auto-tool-choice` | OpenAI-style tool calling |
| `--tool-call-parser` | `hermes`, `llama3_json`, `pythonic`, `qwen3_coder`, `openai`, … |
| `--reasoning-parser` | `deepseek_r1`, `qwen3`, … (reasoning in OpenAI format) |
| `--limit-mm-per-prompt` | `'{"image": 4, "video": 1}'` (default 999 each) |
| `--allowed-local-media-path` / `--allowed-media-domains` | media fetch allowlists |
| `--language-model-only` | disable all multimodal inputs |
| `--enable-lora` / `--lora-modules` | multi-LoRA serving (`name=path`, comma-list) |
| `--max-loras` / `--max-lora-rank` / `--max-cpu-loras` | LoRA capacity |

## Observability

| Flag | Notes |
|---|---|
| `--disable-log-stats` | turn off the periodic stats log |
| `--enable-per-request-metrics` | server-side per-request attribution (queue/prefill/decode) |
| `--kv-cache-metrics` | KV residency metrics (lifetime, reuse gaps; sampled) |
| `--enable-mfu-metrics` | model FLOPs utilization |
| `--otlp-traces-endpoint` | OpenTelemetry traces |
| `--enable-log-requests` | log raw prompts/outputs (privacy!) |

## Config file & misc

| Flag | Notes |
|---|---|
| `--config` | YAML with long-form names (`tensor-parallel-size: 2`) |
| `--json-arg` | JSON blobs; `--json-arg.key value`, `--json-arg.list+ item` |
| `--api-server-count`, `-asc` | frontend processes (default = DP size) |
| `--headless` | worker-only (multi-node DP) |
| `--grpc` | gRPC frontend instead of HTTP |
| `--shutdown-timeout` | graceful shutdown seconds |
| `--seed` | RNG seed |

## Canonical one-liners

```bash
# minimal chat server
vllm serve Qwen/Qwen2.5-7B-Instruct

# production-ish: 32k ctx, key, named model
vllm serve Qwen/Qwen2.5-7B-Instruct --served-model-name qwen7 \
  --max-model-len 32k --api-key sk-local --gpu-memory-utilization 0.9

# 72B AWQ on 2 GPUs
vllm serve Qwen/Qwen2.5-72B-Instruct-AWQ -tp 2 -q awq --dtype half --max-model-len 32768

# long context, quantized KV
vllm serve <model> --max-model-len 128k --kv-cache-dtype fp8

# n-gram speculation (no extra weights)
vllm serve <model> --spec-method ngram --spec-tokens 3

# MoE with expert parallel
vllm serve <moe-model> -tp 2 -ep
```

## Gotchas

- `--api-key` ≠ full auth (`/invocations`, `/metrics` stay open).
- `--max-model-len` OOM at load → lower it or `--kv-cache-dtype fp8`.
- `--gpu-memory-utilization` is a ceiling; two instances on one GPU = 0.5 each.
- AWQ → `--dtype half`; FP8 → `auto`/`bfloat16`.
- Integer suffixes: `1k`=1000, `1K`=1024, `25.6k`=25600.
- YAML config uses long-form names; CLI short forms (`-tp`) don't exist there.
