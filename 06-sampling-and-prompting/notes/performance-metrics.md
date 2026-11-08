# Performance metrics — TTFT, tokens/sec, and how to measure

**Summary.** Local LLM inference has two phases with different bottlenecks: **prefill** (prompt processing — the whole input in parallel, compute-bound, fast) and **decode** (token generation — one token at a time, memory-bandwidth-bound, slow). The metrics that matter: **TTFT** (time to first token = prefill + queue), **TPOT** (time per output token), **ITL** (inter-token latency, the jitter), **E2EL** (end to end), plus raw **prompt tok/s** and **decode tok/s**. Measure with `llama-bench` (offline), `vllm bench serve` (online load), Ollama's `--verbose`, or plain `curl` timing. One-page version: `cheatsheets/perf-metrics.md`. Full tooling deep-dives: [04-llama-cpp/notes/benchmarking.md](../../04-llama-cpp/notes/benchmarking.md) and [05-vllm/notes/benchmarking.md](../../05-vllm/notes/benchmarking.md).

## The two phases

```text
request ──→ [ PREFILL: process N prompt tokens in batches ] ──→ first token ──→ [ DECODE: 1 token/step until EOS ]
              ↑ compute-bound, parallel, ~1000s tok/s            ↑ TTFT ends here      ↑ bandwidth-bound, ~10-100 tok/s
```

(Concepts: [00-foundations/notes/inference-pipeline.md](../../00-foundations/notes/inference-pipeline.md).)

- **Prefill** speed depends on: model size, GPU compute, batch size, quant, flash-attn.
- **Decode** speed depends on: memory *bandwidth* (how fast weights are read per token), quant (bytes/param), how much is on GPU vs CPU, KV cache location.
- A long system prompt makes TTFT big; a big model makes decode slow. They're separate problems with separate fixes.

## Metric definitions

| Metric | Definition | What it tells you |
|---|---|---|
| **TTFT** | request sent → first streamed token | prefill time + queue wait + cold-start effects; the "feels fast?" number |
| **TPOT** | (E2EL − TTFT) / (output tokens − 1) | average decode speed per request (ms/token) |
| **ITL** | gap between consecutive output tokens | decode *jitter*; spikes = preemption, batching, CPU offload |
| **E2EL** | request sent → final token | total user wait |
| **Prompt tok/s** | prompt tokens / prefill time | prefill throughput (llama-bench `pp`) |
| **Decode tok/s** | generated tokens / decode time | reading speed (llama-bench `tg`) |
| **Throughput (tok/s, aggregate)** | all tokens / wall time across all requests | serving capacity under load — the vLLM number |

Relationships: `decode tok/s ≈ 1000 / TPOT(ms)` for a single stream; aggregate throughput > single-stream speed because servers batch concurrent requests.

## Rough "normal" numbers (CUDA, Q4_K_M, single stream)

Ballpark only — bandwidth/compute vary hugely by card. Use as sanity checks, not targets:

| Model | Prompt tok/s (RTX 4090-class) | Decode tok/s (RTX 4090-class) |
|---|---|---|
| 0.6–1.5B | 3k–10k | 80–150 |
| 7–8B | 1k–4k | 40–70 |
| 14B | 800–2k | 25–45 |
| 27B | 400–1k | 15–35 |
| 70B (2–4 GPUs) | 200–600 | 8–20 |

If your numbers are an order of magnitude below this: the model is on the CPU (check `--n-gpu-layers`/`--fit`), the KV cache got pushed to RAM, or you're running a non-CUDA image.

## How to measure, per runtime

### llama.cpp

- **Offline (no server)**: `llama-bench` — the canonical tool. `pp` = prompt processing, `tg` = decode, `pg` = combined. Full guide: [04-llama-cpp/notes/benchmarking.md](../../04-llama-cpp/notes/benchmarking.md).
- **Server logs**: every request logs `prompt eval X tokens Y ms (Z t/s)` and `model eval …` lines — that's your per-request prefill/decode speed.
- **Web UI**: shows prompt/eval t/s per message.
- **Prometheus**: `--metrics` → `/metrics` (token counters, slot states).
- **curl**: time the first byte for TTFT:

```bash
# TTFT ≈ time to first streamed byte
curl -sN -o /dev/null -w 'ttft_first_byte=%{time_starttransfer}s total=%{time_total}s\n' \
  http://localhost:8080/v1/chat/completions -d '{
    "model": "qwen3.8", "stream": true,
    "messages": [{"role": "user", "content": "Write a 200 word story."}]
  }'
```

### Ollama

- **CLI**: `ollama run <model> --verbose "prompt"` prints per-turn stats: `eval count`, `eval duration`, `eval rate` (decode t/s), `load duration`, `prompt eval rate`.
- **Server logs**: same numbers per request (`prompt eval (X tokens) (Y ms) (Z t/s)`).
- **API**: `/api/generate` and `/api/chat` responses include `prompt_eval_count`, `prompt_eval_duration`, `eval_count`, `eval_duration` — compute rates yourself.
- **`/api/ps`**: what's loaded where (GPU/CPU split) — the first thing to check when decode is slow.

```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "qwen3:8b", "prompt": "Write a 200 word story.", "stream": false
}' | python3 -c 'import json,sys; d=json.load(sys.stdin); \
print(f"prompt: {d[\"prompt_eval_count\"]} tok in {d[\"prompt_eval_duration\"]/1e9:.2f}s"); \
print(f"decode: {d[\"eval_count\"]} tok at {d[\"eval_count\"]/(d[\"eval_duration\"]/1e9):.1f} t/s")'
```

### vLLM

- **Online**: `vllm bench serve` — full latency percentiles (TTFT/TPOT/ITL/E2EL) under controlled concurrency. Full guide: [05-vllm/notes/benchmarking.md](../../05-vllm/notes/benchmarking.md).
- **Offline**: `vllm bench throughput` — engine ceiling without HTTP.
- **Prometheus**: `/metrics` (request-level + engine-level; Grafana dashboard in the docs).
- **Per-request**: OpenAI responses include `usage` (token counts); with `--enable-per-request-metrics`, queue vs engine time is attributable.

### Any OpenAI-compatible server (Python)

```python
import time
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8080/v1", api_key="none")
t0 = time.perf_counter()
stream = client.chat.completions.create(
    model="qwen3.8", messages=[{"role": "user", "content": "Write a 200 word story."}],
    stream=True,
)
n = 0
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        if n == 0:
            print(f"TTFT: {time.perf_counter() - t0:.2f}s")
        n += 1
print(f"decode: {n} tokens, {(time.perf_counter() - t0 - ttft):.2f}s")
```

## Reading the numbers

- **High TTFT, fine decode** → long prompt (trim system prompt / RAG context), cold start (first request after load — warm up), or queue (too many concurrent requests; raise `--parallel` or add capacity).
- **Low decode tok/s** → model partially on CPU (`--n-gpu-layers`/`--fit`/`/api/ps`), KV cache on CPU (huge `--ctx-size`), non-quantized weights, or a CPU image.
- **High ITL variance** → continuous batching effects under load (normal), preemption (vLLM: `--gpu-memory-utilization` too low), or CPU offload of some layers.
- **Speculative decoding** (04-llama-cpp) multiplies decode tok/s by the acceptance rate — measure before/after with the same tool.

## Methodology (keeps comparisons honest)

1. **Warm up** — discard the first request after load (lazy init, page cache).
2. **Fix the workload** — same prompt, same lengths, same seed; change one variable at a time.
3. **Single stream vs load** — report which you measured; aggregate throughput only makes sense under concurrency.
4. **Record the command** — numbers are only comparable with exact flags (quant, ctx, parallel, KV dtype).
5. **Match the shape** — serving 32k context? Benchmark at 32k depth, not the 512 default.

## Sources

- [llama.cpp — llama-bench README](https://github.com/ggml-org/llama.cpp/blob/master/tools/llama-bench/README.md)
- [llama.cpp — SPEED-Bench client (server-level)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/bench/speed-bench/README.md)
- [vLLM — Benchmark CLI](https://docs.vllm.ai/en/stable/benchmarking/cli/)
- [vLLM — Metrics design doc](https://docs.vllm.ai/en/stable/design/metrics/)
- [Ollama — API reference (timing fields)](https://docs.ollama.com/api)
- [This repo — 00-foundations/notes/inference-pipeline.md](../../00-foundations/notes/inference-pipeline.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B Q4_K_XL on :8080 (ctx 131k, flash-attn, q4_0 KV): 1k-token prompt → TTFT 0.8s, decode 22 t/s; `llama-bench -d 32768` tg128 = 19 t/s (context depth costs ~15%); Ollama 8B on the same box: prompt 2.1k t/s, decode 61 t/s — the 3.4× model-size ratio shows up almost exactly in decode")
