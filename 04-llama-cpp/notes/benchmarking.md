# Benchmarking with llama-bench

**Summary.** `llama-bench` is the canonical way to compare quants, models, and flags on *your* hardware. It runs three test types — prompt processing (`pp`, batched), text generation (`tg`, sequential decode), and combined (`pg`) — repeats each N times, and prints average t/s ± stddev. It lives in the `full` Docker image. Use it to answer: "does Q5_K_M beat Q4_K_M here?", "what does `--flash-attn on --cache-type-k q4_0` gain me?", "which `--n-gpu-layers` split is fastest?".

## Quick start (Docker)

```bash
# one model, default pp 512 + tg 128
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-bench -m /m/model.gguf

# compare two quants of the same model, decode at 3 lengths
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-bench -m /m/model-Q4_K_M.gguf -m /m/model-Q5_K_M.gguf -p 0 -n 128,256,512

# prompt processing across batch sizes (which --batch-size is best?)
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-bench -m /m/model.gguf -n 0 -p 1024 -b 128,256,512,1024

# flash-attn + KV quant effect on long context (pre-fill 16k, decode 128)
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-bench -m /m/model.gguf -d 16384 -p 0 -n 128 \
  -fa on -ctk q4_0 -ctv q4_0
```

## Key flags

| Flag | Default | Meaning |
|---|---|---|
| `-m, --model` | — | model(s); repeat or comma-separate for multiple |
| `-p, --n-prompt` | 512 | prompt length for `pp`/`pg` |
| `-n, --n-gen` | 128 | tokens to generate for `tg`/`pg` |
| `-pg pp,tg` | — | combined test (e.g. `-pg 2048,256`) |
| `-d, --n-depth` | 0 | pre-fill KV cache to this depth (long-context tests) |
| `-r, --repetitions` | 5 | repeats, averaged |
| `-o, --output` | md | `md`, `csv`, `json`, `jsonl`, `sql` |
| `-b` / `-ub` | 2048 / 512 | batch / ubatch (repeatable for sweeps) |
| `-t, --threads` | auto | CPU threads (repeatable for sweeps) |
| `-ngl` | -1 | GPU layers (sweep `-ngl 0,8,16,31,-1` to find the split) |
| `-fa, --flash-attn` | auto | `on`/`off`/`auto` |
| `-ctk` / `-ctv` | f16 | KV cache dtypes |
| `-sm` / `-ts` / `-mg` | layer / — / 0 | multi-GPU split mode / fractions / main GPU |
| `-ncmoe` | 0 | MoE layers kept on CPU |
| `--no-warmup` | off | skip warmup (faster, slightly noisier) |
| `-hf` | — | benchmark straight from a HF repo |

Any test parameter can take **multiple values** (comma or repeated flag) — llama-bench runs the full cross-product. Ranges: `first-last`, `first-last+step`, `first-last*mult`.

## Reading the output

```
| model                  |   size | params | backend | ngl | test      |        t/s |
| ---------------------- | -----: | -----: | ------- | --: | --------- | ---------: |
| qwen3.8 27B Q4_K_M     | 16.2 G | 27.0 B | CUDA    |  -1 | pp512     |    612.4 ± 8.1 |
| qwen3.8 27B Q4_K_M     | 16.2 G | 27.0 B | CUDA    |  -1 | tg128     |     38.2 ± 0.4 |
```

- **pp** (prompt processing) = tokens/s in batched prefill — what you feel when a long prompt/system context is ingested.
- **tg** (text generation) = tokens/s sequential decode — what you feel while reading.
- ± is stddev across repetitions; > ~5% means noisy run (close other workloads).
- `llama-bench` excludes tokenization and sampling time.

## Methodology that keeps results honest

1. **One variable at a time** — change exactly one flag/model between rows.
2. **Same machine state** — no other GPU work, same clocks (pinned if possible), same container/image.
3. **`-r 5` minimum** (default); bump to 10 for close calls.
4. **Compare at your real shape** — if you serve 32k context, test with `-d 32768`, not the 512 default.
5. **Save the table** — `-o csv -oe json` or just capture stdout into your notes' *My experiments*.
6. For end-to-end serving numbers (incl. HTTP overhead, batching across slots), use the server's `timings` field or the SPEED-Bench client (`tools/server/bench/speed-bench`) instead.

## Gotchas

1. **tg at n=128 vs your real workload** — decode speed can shift with context depth; use `-d` for long-context decisions.
2. **`-ngl -1` is the default here** (unlike the server's `auto`) — be explicit when comparing CPU/GPU splits.
3. **First run is cold** — the default warmup handles it; `--no-warmup` is for speed, not accuracy.
4. **MoE models** — `tg` speed is mostly CPU/DRAM-bound on experts; `--n-cpu-moe` changes the picture, include it in the sweep.
5. **Don't compare pp across batch sizes as "speed"** — bigger batch = higher t/s by construction; compare *time for the same prompt* or hold `-b` fixed.

## Sources

- [llama-bench README](https://github.com/ggml-org/llama.cpp/blob/master/tools/llama-bench/README.md)
- [SPEED-Bench client (server-level)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/bench/speed-bench/README.md)

## My experiments

- (fill in: e.g. "llama-bench on RTX 5090: Q4_K_M tg128 = 38.2 t/s vs Q5_K_M = 33.1 t/s (−13%); `--flash-attn on --cache-type-k q4_0` at `-d 32768`: pp 401 → 512 t/s; best ngl split for 27B = −1 (all layers)")
