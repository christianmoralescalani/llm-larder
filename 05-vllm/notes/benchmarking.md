# Benchmarking with `vllm bench`

**Summary.** `vllm bench` is vLLM's built-in load tester — no external tooling needed. The workhorse is `vllm bench serve`: it hammers a *running* server with a realistic workload and reports throughput + latency percentiles (TTFT, TPOT, ITL, E2EL). Companions: `vllm bench throughput` (max offline throughput, no server), `vllm bench latency` (engine-level), `vllm bench forward` (exact-batch model timing). In Docker, run the client inside the same image with `--entrypoint vllm`. Use it to answer: "what concurrency does this box sustain?", "does prefix caching actually help my workload?", "what did that flag change buy?".

## Online benchmark (`vllm bench serve`)

```bash
# server first (Docker)
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest --model Qwen/Qwen2.5-7B-Instruct

# benchmark client (same image, entrypoint override)
docker run --rm --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest bench serve \
  --base-url http://host.docker.internal:8000 \
  --model Qwen/Qwen2.5-7B-Instruct \
  --endpoint /v1/chat/completions \
  --dataset-name random \
  --random-input-len 512 --random-output-len 128 \
  --num-prompts 100 \
  --max-concurrency 16 \
  --save-result --result-dir ./bench
```

### Workload knobs

| Flag | Default | What it does |
|---|---|---|
| `--dataset-name` | `random` | `random`, `sharegpt`, `hf` (any HF dataset), `custom` (jsonl), `custom_audio`, `custom_image`, `random-mm`, `mt_bench`, `spec_bench`, `speed_bench`, … |
| `--random-input-len` / `--random-output-len` | 1024 / 128 | synthetic prompt/output sizes |
| `--random-range-ratio` | 1.0 | ± spread around the lengths (`0.5` = half to 1.5×) |
| `--num-prompts` | — | total requests |
| `--request-rate` | `inf` | req/s; `inf` = fire everything at once (burst); a number = Poisson arrival |
| `--max-concurrency` | — | cap in-flight requests (the main "how many users" dial) |
| `--num-warmups` | 0 | warmup requests excluded from stats |
| `--temperature` / `--top-p` / `--top-k` | 0 / 1.0 / — | sampling for the benchmark |
| `--ignore-eos` | off | force full `--random-output-len` (steady output length) |
| `--seed` | — | reproducibility |
| `--routing zipf` | — | MoE: skew prompt distribution (real-world hot prefixes) |
| `--probe-request-rate` | — | send small probe requests alongside; measures interference on unrelated traffic |

### Reading the output

```
============ Serving Benchmark Result ============
Successful requests:                     100
Benchmark duration (s):                  5.78
Total input tokens:                      1369
Total generated tokens:                  2212
Request throughput (req/s):              1.73
Output token throughput (tok/s):         382.89
Total token throughput (tok/s):          619.85
---------------Time to First Token----------------
Mean TTFT (ms):                          71.54
Median TTFT (ms):                          73.88
P99 TTFT (ms):                            79.49
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                           7.91
---------------Inter-token Latency----------------
Mean ITL (ms):                            7.74
==================================================
```

| Metric | Meaning |
|---|---|
| **TTFT** | send request → first streamed token (prefill + queue time) |
| **TPOT** | (E2EL − TTFT) / (output tokens − 1) — average decode speed per request |
| **ITL** | gap between consecutive output tokens (jitter; spikes = preemption/batching effects) |
| **E2EL** | send request → final token |
| **Output token throughput** | aggregate decode t/s across all requests — *the* serving number |

`--percentile-metrics ttft,tpot,itl,e2el` + `--metric-percentiles 50,90,99` control which percentiles print. `--save-result` writes JSON; `--save-detailed` adds per-request rows (pair with server-side `--enable-per-request-metrics` to attribute latency to queue vs engine).

## Offline throughput (`vllm bench throughput`)

No server — measures the engine's ceiling directly:

```bash
docker run --rm --gpus all --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest bench throughput \
  --model Qwen/Qwen2.5-7B-Instruct \
  --dataset-name random --random-input-len 500 --random-output-len 150 \
  --num-prompts 100
# → Throughput: 7.15 requests/s, 4647.50 total tokens/s, 1072.50 output tokens/s
```

Use for: "what's the max this hardware+model does?" and A/B testing flags (`--kv-cache-dtype fp8`, `--enforce-eager` off/on, speculative config) without HTTP in the loop.

## Latency & forward

- `vllm bench latency` — engine-level latency at fixed batch sizes (no server, no load model).
- `vllm bench forward` — exact-batch model timing, including steady-state pipeline-parallel forwards (kernel-level).
- `vllm bench sweep serve` — repeated `bench serve` runs that **reset server caches between runs** (when you don't want prefix-cache reuse skewing the comparison).
- `vllm bench mm-processor` — multimodal preprocessing speed (image/video tokenization cost).

## Methodology that keeps results honest

1. **Warm up** — first requests after boot include lazy init; use `--num-warmups` or discard the first run.
2. **Fix the workload** — same `--seed`, same lengths, same concurrency across runs; change one variable at a time.
3. **Burst vs sustained** — `--request-rate inf` measures peak; a finite rate measures steady-state. Report which you used.
4. **Prefix caching confounds** — repeated prompts get cached (faster TTFT). `vllm bench sweep serve` or fresh seeds if you're comparing raw speed.
5. **Concurrency sweep** — run `--max-concurrency 1, 4, 16, 64, 256` to find the knee (where TTFT/TPOT start climbing) — that's your serving capacity.
6. **Production-grade load** — the docs point to [GuideLLM](https://github.com/guidellm/guidellm) for production server benchmarking (SLO-aware, more traffic models).
7. **Record the command** — `--save-result --label "tp2-awq-fp8kv"`; results are only comparable with the exact flags.

## Gotchas

1. **`--base-url` in Docker** — the client container needs to reach the server container: `host.docker.internal` (host network), a compose network, or `--network host`.
2. **`--model` must be the served name** — if the server used `--served-model-name`, pass that here or requests 404.
3. **`--ignore-eos` changes the workload** — forced output length removes natural early stops; fine for steady-state, misleading for "real" latency.
4. **TPOT ≠ ITL** — TPOT is an average over the request; ITL shows the distribution. High mean ITL with low TPOT = uneven batching.
5. **Throughput number ≠ your workload** — `random` data is uniform; real traffic has hot prefixes (prefix-cache friendly) and long-tail lengths. ShareGPT/HF datasets are closer to real.
6. **Benchmark the server you'll actually run** — `--enforce-eager`, `--max-model-len`, and KV dtype all move the numbers.

## Sources

- [Benchmark CLI](https://docs.vllm.ai/en/stable/benchmarking/cli/)
- [vllm bench serve reference](https://docs.vllm.ai/en/stable/cli/bench/serve/)
- [GuideLLM](https://github.com/guidellm/guidellm)
- [Performance metrics (06 section, this repo)](../../Structure.md)

## My experiments

- (fill in: e.g. "7B on RTX 5090: concurrency sweep 1→256, knee at 32 (TPOT 9ms → 25ms); prefix caching on: TTFT 180ms → 60ms on repeated system prompt; `vllm bench throughput` ceiling: X tok/s output")
