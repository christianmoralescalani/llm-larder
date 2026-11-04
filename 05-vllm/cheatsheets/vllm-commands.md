# Cheatsheet — vLLM commands

**TL;DR.** One binary, `vllm`, with subcommands: `serve` (the server — the image's default entrypoint), `chat` / `complete` (client CLIs), `bench` (load testing), `run-batch` (batched offline), `preload` (weight preloading), `download-kernels` (preinstall kernels), `collect-env` (debug dump). In Docker, override the entrypoint to use anything but `serve`. Verified against vLLM v0.32.0 (stable, 2026-10-18), 2026-11-03.

## Command table

| Command | Job | In Docker |
|---|---|---|
| `vllm serve <model>` | OpenAI-compatible HTTP server (default entrypoint) | args after the image tag |
| `vllm chat` | interactive chat client (against a running server) | `--entrypoint vllm` |
| `vllm complete` | one-shot completion client | `--entrypoint vllm` |
| `vllm bench serve` | online load test (throughput + TTFT/TPOT/ITL percentiles) | `--entrypoint vllm` |
| `vllm bench throughput` | offline max throughput (no server) | `--entrypoint vllm` |
| `vllm bench latency` | engine-level latency | `--entrypoint vllm` |
| `vllm bench forward` | exact-batch model timing (incl. PP) | `--entrypoint vllm` |
| `vllm bench sweep serve` | repeated serve runs, cache reset between | `--entrypoint vllm` |
| `vllm bench mm-processor` | multimodal preprocessing speed | `--entrypoint vllm` |
| `vllm run-batch` | offline batched inference (JSONL in/out) | `--entrypoint vllm` |
| `vllm preload` | preload weights into shared cache (`ipc_cache` load format) | `--entrypoint vllm` |
| `vllm download-kernels` | preinstall FlashInfer kernels (faster startup) | pip install only |
| `vllm collect-env` | print environment info (debugging) | `--entrypoint vllm` |
| `python` (offline `LLM`) | in-process batched inference | `--entrypoint python` |

## Day-to-day

```bash
# serve (Docker)
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest --model Qwen/Qwen2.5-7B-Instruct

# health & models
curl http://localhost:8000/health
curl http://localhost:8000/v1/models

# interactive chat client
docker run --rm -it --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest chat --url http://localhost:8000/v1

# one-shot completion
docker run --rm --entrypoint vllm \
  vllm/vllm-openai:latest complete --url http://localhost:8000/v1 \
  --model Qwen/Qwen2.5-7B-Instruct "San Francisco is a"

# quick load test
docker run --rm --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest bench serve \
  --base-url http://host.docker.internal:8000 \
  --model Qwen/Qwen2.5-7B-Instruct \
  --endpoint /v1/chat/completions \
  --dataset-name random --random-input-len 512 --random-output-len 128 \
  --num-prompts 100 --max-concurrency 16

# offline Python (entrypoint override)
docker run --rm --gpus all --entrypoint python \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -v /tmp/gen.py:/gen.py:ro \
  vllm/vllm-openai:latest /gen.py

# debug: what does the environment look like?
docker run --rm --entrypoint vllm vllm/vllm-openai:latest collect-env

# help & flag search
vllm serve --help=all
vllm serve --help=max          # search flags containing "max"
vllm serve --help=ModelConfig  # one argument group
```

## Client CLIs vs SDK

| Need | Use |
|---|---|
| Quick human check | `vllm chat` / `vllm complete` (`--stats` for TTFT/tok/s) |
| App integration | any OpenAI SDK → `base_url=http://host:8000/v1` |
| Load testing | `vllm bench serve` |
| Batch jobs | `vllm run-batch` or offline `LLM` |
| CI smoke test | `curl /health` + one `/v1/chat/completions` |

## Gotchas

- `--entrypoint vllm` (not `python`) for the client tools; `--entrypoint python` for offline scripts.
- `bench serve --model` must equal the server's `--served-model-name`.
- `--base-url` from another container: `host.docker.internal` (host net) or a compose network.
- `vllm chat --quick "hi"` = one-shot without a REPL.
- `run-batch` wants JSONL (`{"prompt": ...}` per line) — see docs for the full schema.
- `collect-env` output is what you paste into bug reports.
