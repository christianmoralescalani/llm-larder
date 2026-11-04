# `vllm serve` — key flags

**Summary.** `vllm serve` takes ~200 flags (groups: Frontend, ModelConfig, CacheConfig, SchedulerConfig, ParallelConfig, …); you'll touch maybe 12. The ones that matter: model identity (`--model`, `--served-model-name`), network (`--host`, `--port`, `--api-key`), memory (`--gpu-memory-utilization`, `--max-model-len`, `--kv-cache-dtype`), throughput (`--max-num-seqs`, `--max-num-batched-tokens`), precision (`--dtype`, `--quantization`), speedups (`--enable-prefix-caching`, `--speculative-config`), and scale (`--tensor-parallel-size`, `--pipeline-parallel-size`, `--data-parallel-size`). Full reference: `cheatsheets/vllm-serve-flags.md` and `vllm serve --help=all` (also `--help=max` to search).

## Model & identity

| Flag | Default | Notes |
|---|---|---|
| `--model` | `Qwen/Qwen3-0.6B` | HF repo ID **or local path** (safetensors *or* GGUF). Also the metrics `model_name` tag |
| `--served-model-name` | = `--model` | name(s) clients use in the `model` field; comma-list allowed, first wins in responses |
| `--tokenizer` | = model | separate tokenizer (rare) |
| `--trust-remote-code` | off | run custom code from HF repos |
| `--hf-token` | — | gated repos (`HF_TOKEN` env also works) |
| `--revision` | — | pin branch/tag/commit |
| `--load-format` | `auto` | `safetensors`, `dummy` (profiling), `tensorizer`, `runai_streamer`, `sharded_state`, `ipc_cache` (with `vllm preload`), … |

## Network & auth

| Flag | Default | Notes |
|---|---|---|
| `--host` | `0.0.0.0` | |
| `--port` | `8000` | |
| `--uds` | — | Unix socket path (host/port ignored) |
| `--api-key` | none | comma-separated keys (rotation); **only** guards `/v1`, `/v2`, `/inference` — not `/invocations`, `/metrics` |
| `--ssl-certfile` / `--ssl-keyfile` | — | HTTPS |
| `--allowed-origins` | `[]` | CORS (browser access disabled by default) |

## Memory & context

| Flag | Default | Notes |
|---|---|---|
| `--gpu-memory-utilization` | `0.92` | fraction of GPU VRAM the instance may use (also `--device-memory-utilization`); per-instance — two instances on one GPU can each claim 0.5 |
| `--kv-cache-memory-bytes` | auto | explicit KV size; overrides `--gpu-memory-utilization` when set |
| `--max-model-len` | model max / fit | prompt+output cap; `32k`/`32K`/`25.6k` suffixes; `-1`/`auto` = largest that fits |
| `--kv-cache-dtype` | `auto` | `fp8`, `fp8_e5m2`, `int8_per_token_head`, `nvfp4`, … — halves/quarters KV VRAM for long contexts |
| `--block-size` | 16 | KV block size in tokens |
| `--enable-prefix-caching` | **on** (V1) | hash-based KV block reuse for shared prefixes; `--disable-prefix-caching` to turn off |

## Throughput & scheduling

| Flag | Default | Notes |
|---|---|---|
| `--max-num-seqs` | engine-derived (typically 256) | max concurrent sequences in one iteration — the main throughput dial |
| `--max-num-batched-tokens` | 8192 (engine-derived) | max tokens per iteration (prefill+decode combined) |
| `--enable-chunked-prefill` | **on** (V1) | slice long prefills across iterations so decode isn't starved |
| `--scheduling-policy` | `fcfs` | `priority` = lower number served first |
| `--watermark` | `0.0` | keep this fraction of KV blocks free to avoid preemption thrash |

## Precision & quantization

| Flag | Default | Notes |
|---|---|---|
| `--dtype` | `auto` | `auto` = FP16 for FP32/FP16 models, BF16 for BF16; `bfloat16`, `float16`, `float32` |
| `--quantization`, `-q` | auto-detect | `awq`, `gptq`, `fp8`, `bitsandbytes`, `gguf`, `compressed-tensors`, `modelopt`, `torchao`, …; if unset, reads `quantization_config` from the model repo |
| `--enforce-eager` | off | disable torch.compile + CUDA graphs (debugging; slower) |
| `--attention-backend` | auto | `FLASH_ATTN`, `FLASHINFER` (NVIDIA), ROCm/XPU variants |

Quantized models usually need no flag — the repo's `quantization_config` drives it. See `notes/model-support.md` for the format matrix.

## Speculative decoding

| Flag | Notes |
|---|---|
| `--speculative-config`, `-sc` | JSON: `'{"model":"<draft>","method":"eagle3","num_speculative_tokens":3}'` |
| `--spec-method` | `eagle`, `eagle3`, `mtp`, `dflash`, `dspark`, `ngram`, `ngram_gpu`, `suffix`, `draft_model`, … (auto-detected when `model` is given) |
| `--spec-model` | draft model / EAGLE head / MTP weights |
| `--spec-tokens` | draft tokens per step |

Methods: model-based (EAGLE, MTP, draft model, DFlash/DSpark) give the biggest latency wins; `ngram`/`suffix` need no extra weights (great for code/boilerplate). See `notes/what-is-vllm.md` and the [speculative decoding docs](https://docs.vllm.ai/en/stable/features/speculative_decoding/).

## Parallelism (single node)

| Flag | Default | Notes |
|---|---|---|
| `--tensor-parallel-size`, `-tp` | 1 | split each layer across N GPUs; needs `--ipc=host` in Docker |
| `--pipeline-parallel-size`, `-pp` | 1 | split layers across stages (multi-node: pp = nodes) |
| `--data-parallel-size`, `-dp` | 1 | N independent replicas behind one endpoint |
| `--enable-expert-parallel`, `-ep` | off | MoE: shard experts across GPUs instead of TP |

Decision tree: fits on 1 GPU → no flags. Fits on a node → `-tp N`. Bigger than a node → `-tp <gpus/node> -pp <nodes>` (plus `--nnodes`/`--node-rank`). See `notes/multi-gpu.md`.

## Chat & tools

| Flag | Notes |
|---|---|
| `--chat-template` | override the tokenizer's template (file or inline Jinja) |
| `--default-chat-template-kwargs` | e.g. `'{"enable_thinking": false}'` for Qwen3/DeepSeek thinking models |
| `--enable-auto-tool-choice` + `--tool-call-parser` | OpenAI-style tool calling (`hermes`, `llama3_json`, `pythonic`, `qwen3_coder`, …) |
| `--reasoning-parser` | surface reasoning content in OpenAI format (`deepseek_r1`, `qwen3`, …) |
| `--enable-lora` + `--lora-modules name=path` | multi-LoRA serving (`--max-loras`, `--max-lora-rank`) |

## Observability

| Flag | Notes |
|---|---|
| `--disable-log-stats` | turn off the periodic stats log (Prometheus `/metrics` is separate and always on) |
| `--enable-per-request-metrics` | server-side per-request attribution (queue/prefill/decode) |
| `--kv-cache-metrics` | KV cache residency metrics (lifetime, reuse gaps; sampled) |
| `--enable-mfu-metrics` | model FLOPs utilization |
| `--otlp-traces-endpoint` | OpenTelemetry traces (+ `--collect-detailed-traces all\|model\|worker`) |
| `--enable-log-requests` | log raw prompts/outputs (privacy!) |

## Config file

`--config config.yaml` loads long-form args from YAML (see `notes/docker.md`); `--json-arg '{...}'` for JSON blobs (nested keys via `--json-arg.key value`, list append via `+`).

## Canonical examples

```bash
# 8B chat model, 32k context, prefix caching explicit
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-7B-Instruct \
  --served-model-name qwen7 \
  --max-model-len 32k --enable-prefix-caching \
  --api-key sk-local

# 70B on 2 GPUs, AWQ 4-bit
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-72B-Instruct-AWQ \
  --tensor-parallel-size 2 \
  --quantization awq --dtype half \
  --max-model-len 32768

# n-gram speculation, no extra model
  --spec-method ngram --spec-tokens 3
```

## Gotchas

1. **`--max-model-len` vs VRAM** — bigger context = bigger KV cache; if you OOM at load, lower it (or `--kv-cache-dtype fp8`).
2. **`--gpu-memory-utilization` is a ceiling, not a target** — vLLM allocates KV blocks up to the ceiling; two vLLMs on one GPU need manual splitting (0.5 each).
3. **`--dtype half` for AWQ** — AWQ quants generally want FP16 activations; `auto` usually gets this right, but `half` is the documented pairing.
4. **`--speculative-config` JSON quoting** — inside `docker run` the single quotes survive; in compose `command:` lists, pass it as one list item.
5. **`--api-key` ≠ full auth** — see the network table; put a reverse proxy in front for anything public.
6. **`--enforce-eager` changes numbers** — benchmark with it off (CUDA graphs); use it only to debug OOMs/crashes.
7. **Flag names are long-form in YAML** — `--tensor-parallel-size` → `tensor-parallel-size:` in `config.yaml`.

## Sources

- [vllm serve — full CLI reference](https://docs.vllm.ai/en/stable/cli/serve/)
- [Server Arguments (YAML)](https://docs.vllm.ai/en/stable/configuration/serve_args/)
- [Optimization & tuning](https://docs.vllm.ai/en/stable/configuration/optimization/)
- [Speculative decoding](https://docs.vllm.ai/en/stable/features/speculative_decoding/)

## My experiments

- (fill in: e.g. "7B on RTX 5090: `--max-num-seqs 256` vs `64` — aggregate t/s X vs Y; `--kv-cache-dtype fp8` fit 64k context where bf16 KV OOM'd; prefix caching hit rate ~Z% on repeated system prompt")
