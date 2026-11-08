# Cheatsheet — performance metrics

**TL;DR.** Two phases: **prefill** (prompt in, parallel, fast) and **decode** (one token out per step, bandwidth-bound, slow). The numbers: **TTFT** (send → first token = prefill + queue), **TPOT** (ms per output token), **ITL** (jitter between tokens), **E2EL** (total), plus raw **prompt tok/s** and **decode tok/s**. Measure with `llama-bench` (offline), `vllm bench serve` (online), Ollama `--verbose`, or `curl -w`. Deep dive: `notes/performance-metrics.md`.

## Metric definitions

| Metric | Definition | Feels like |
|---|---|---|
| TTFT | request → first streamed token | "how long until it starts typing" |
| TPOT | (E2EL − TTFT) / (tokens − 1) | average reading speed |
| ITL | gap between consecutive tokens | stutter/jitter |
| E2EL | request → final token | total wait |
| prompt tok/s | prompt tokens / prefill time | long-prompt ingestion speed |
| decode tok/s | output tokens / decode time | reading speed (`≈ 1000/TPOT`) |
| throughput (aggregate) | all tokens / wall time, all requests | serving capacity under load |

## Measure it

```bash
# llama.cpp — offline benchmark (no server): pp = prefill, tg = decode
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-bench -m /m/model.gguf -p 1024 -n 128 -r 5

# llama.cpp server — TTFT via curl (time to first byte)
curl -sN -o /dev/null -w 'ttft=%{time_starttransfer}s total=%{time_total}s\n' \
  http://localhost:8080/v1/chat/completions \
  -d '{"model":"qwen3.8","stream":true,"messages":[{"role":"user","content":"Write a 200 word story."}]}'

# llama.cpp server — per-request speeds: just read the logs
#   "prompt eval 512 tokens (123 ms) (4162.5 t/s)"   ← prefill
#   "model eval 128 tokens (5.4 s) (23.7 t/s)"       ← decode

# Ollama — per-turn stats
ollama run qwen3:8b --verbose "Write a 200 word story."
# → eval rate (decode t/s), prompt eval rate (prefill t/s), load duration

# Ollama — API timing fields
curl -s http://localhost:11434/api/generate -d '{"model":"qwen3:8b","prompt":"hi","stream":false}' \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); \
print(f"prompt {d[\"prompt_eval_count\"]} tok @ {d[\"prompt_eval_count\"]/(d[\"prompt_eval_duration\"]/1e9):.0f} t/s, \
decode {d[\"eval_count\"]} tok @ {d[\"eval_count\"]/(d[\"eval_duration\"]/1e9):.1f} t/s")'

# vLLM — online load test with latency percentiles
docker run --rm --entrypoint vllm -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest bench serve \
  --base-url http://host.docker.internal:8000 --model <served-name> \
  --endpoint /v1/chat/completions --dataset-name random \
  --random-input-len 512 --random-output-len 128 --num-prompts 100 --max-concurrency 16
```

## Rough "normal" (Q4_K_M, single stream, RTX 4090-class)

| Model | prompt tok/s | decode tok/s |
|---|---|---|
| 0.6–1.5B | 3k–10k | 80–150 |
| 7–8B | 1k–4k | 40–70 |
| 14B | 800–2k | 25–45 |
| 27B | 400–1k | 15–35 |
| 70B (2–4 GPUs) | 200–600 | 8–20 |

An order of magnitude below → model on CPU, KV cache on RAM, or wrong (non-CUDA) image.

## Diagnose

| Symptom | Check |
|---|---|
| High TTFT | long prompt (trim), cold start (warm up), queue (raise `--parallel`) |
| Low decode tok/s | `--n-gpu-layers`/`/api/ps` (CPU offload?), `--ctx-size` (KV on CPU?), quant, image |
| High ITL variance | batching under load (normal), preemption (vLLM `--gpu-memory-utilization`), partial offload |
| Slow everything | wrong image (no CUDA), thermal throttling, other GPU work |

## Methodology

1. Warm up — discard the first request after load.
2. Fix the workload — same prompt/lengths/seed; one variable at a time.
3. Report single-stream vs loaded (concurrency) — different numbers.
4. Record the exact command/flags — numbers are only comparable with them.
5. Match context depth to your real serving shape (`-d 32768`, not the 512 default).

Verified against llama.cpp master (v0.8.0), Ollama 0.32.x, vLLM 0.9.x — 2026-11-08.
