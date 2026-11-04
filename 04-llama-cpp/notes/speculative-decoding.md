# Speculative decoding

**Summary.** Decode is memory-bandwidth bound: generating N tokens one-by-one wastes the GPU. Speculative decoding fixes that — a cheap *draft* mechanism proposes N tokens, the target model verifies the whole batch in one forward pass, and accepted tokens are kept. Net speedup ≈ acceptance rate × batch efficiency, so it helps most when (a) the draft is good (high acceptance) and (b) the target is big/slow per token. llama.cpp implements it in `llama-server` and `llama-cli` via `--spec-type`, with five families: draft model, EAGLE-3, DFlash/DSpark, MTP heads, and n-gram lookups (no extra model).

## The idea in one paragraph

Prompt processing (many tokens, one pass) is far more efficient per token than generation (one token, one pass). So: draft k tokens cheaply → target scores all k+1 positions in a single batch → keep the longest accepted prefix, resample the first rejected position from the target's distribution. Output distribution is **identical** to plain sampling (rejection sampling), so quality doesn't change — only speed. If acceptance is low, you paid for extra forward work and get slower, not faster.

## The implementations (`--spec-type`)

| Type | What drafts | Extra model? | Notes |
|---|---|---|---|
| `draft-simple` | small draft GGUF (`-md` / `--spec-draft-model`, or `-hfd` from HF) | yes | classic approach; draft shares the target's tokenizer |
| `draft-eagle3` | EAGLE-3 head that reads the target's hidden states | yes (tiny, 1-layer) | higher acceptance than a standalone draft of the same size; convert with `--target-model-dir` |
| `draft-dflash` | block-diffusion model, whole block per step | yes (small, multi-layer) | GPU-friendly; `--spec-draft-n-max` clamps to trained block size; pair with `-fa on` |
| `draft-dspark` | DFlash backbone + semi-autoregressive Markov head | yes | newest variant |
| `draft-mtp` | **MTP heads already inside the target model** | no | e.g. Qwen3.8 ships an MTP draft — zero extra weights, just `--spec-type draft-mtp` |
| `ngram-simple` | patterns already in the context | no | look back N tokens, find a prior match, draft M |
| `ngram-map-k` / `ngram-map-k4v` | n-gram key → m-gram value map | no | better than simple; k4v keeps up to 4 candidate values (experimental) |
| `ngram-mod` | shared-pool n-gram hasher | no | stats-based, good for repetitive/code workloads |
| `ngram-cache` | running n-gram statistics (+ external stat files) | no | most statistical of the bunch |

Types are comma-separated and **can be mixed** (e.g. `--spec-type ngram-mod,ngram-map-k4v`, or an n-gram type + a draft model).

## Quick starts (Docker)

```bash
# MTP: the target model carries its own draft heads (e.g. Qwen3.8 + MTP draft GGUF)
docker run -d --name llm -p 8080:8080 --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/Qwen3.8-27B-UD-Q4_K_XL.gguf \
  --spec-type draft-mtp --spec-draft-model /models/mtp-Qwen3.8-27B-Q4_0.gguf \
  --spec-draft-n-max 2 --host 0.0.0.0 --port 8080

# classic small draft model
docker run -d --name llm -p 8080:8080 --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/big-model.gguf \
  --spec-type draft-simple --spec-draft-model /models/draft-0.5B.gguf \
  --spec-draft-n-max 4 --host 0.0.0.0 --port 8080

# no extra model at all: n-gram speculation (great for code editing / boilerplate)
docker run -d --name llm -p 8080:8080 --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --spec-type ngram-simple \
  --host 0.0.0.0 --port 8080
```

## Key flags

| Flag | Default | Meaning |
|---|---|---|
| `--spec-type` | `none` | comma-separated list of the types above |
| `--spec-draft-model, -md` | — | draft GGUF (or `-hfd` for HF repo) |
| `--spec-draft-n-max` | 3 | max draft tokens per step (higher = bigger batch, more wasted work if rejected) |
| `--spec-draft-n-min` | 0 | min draft tokens |
| `--spec-draft-p-min` | 0.00 | greedy acceptance threshold (draft token must beat this prob) |
| `--spec-draft-sampling` | `greedy` | `greedy` (argmax) vs `probabilistic` (rejection-sampled draft) |
| `--spec-draft-ngl, -ngld` | auto | draft model VRAM offload |
| `--spec-draft-device, -devd` | follows `--device` | put the draft on a different GPU |
| `--spec-draft-n-ubatch` | = `--ubatch-size` | draft's physical batch (lower it to spare VRAM) |
| `--spec-draft-type-k/v` | f16 | draft's KV cache dtype |
| `--spec-ngram-{simple,map-k,map-k4v}-size-n/m` | 12 / 48 | lookup n-gram length / draft m-gram length |
| `--spec-ngram-*-min-hits` | 1 | required prior hits before drafting |
| `--spec-ngram-mod-n-min/n-max/n-match` | 48/64/24 | ngram-mod window sizes |
| `--spec-synth-len` / `--spec-synth-rates` | — | synthetic acceptance for benchmarking only |

## Reading the stats

The server prints acceptance stats per implementation:

```
draft acceptance rate = 0.70312 (  90 accepted /  128 generated)
statistics ngram_mod: #calls = 810, #gen drafts = 15, #acc drafts = 15, #gen tokens = 960, #acc tokens = 730
```

- **acceptance rate** = accepted/generated draft tokens. > ~0.6 is a good sign of real speedup; < ~0.3 and you're paying overhead for nothing.
- `#gen tokens` vs `#acc tokens` shows the same ratio per implementation.
- End-to-end comparison: run `llama-bench` (or the SPEED-Bench client in `tools/server/bench/speed-bench`) with and without `--spec-type` on identical prompts.

## When it helps / hurts

**Helps:**
- Big target models where one decode step is slow (the batched verify amortizes well).
- Repetitive, structured, or code-like output (n-gram methods shine: boilerplate, JSON, refactors).
- MTP/EAGLE-3 drafts trained for your exact model (high acceptance by construction).
- GPU-bound setups — the verify batch uses the GPU efficiently.

**Hurts / doesn't help:**
- Small models on CPU (draft + verify both cheap; overhead dominates).
- Low-acceptance pairings (draft too small, or creative/high-temp output) — you do 2× work for ~1× tokens.
- Memory-tight VRAM: the draft model (or its KV) is extra; use `--spec-draft-n-ubatch`, `--spec-draft-type-k/v q4_0`, or `--spec-draft-device none` to shrink it.
- Prompt-processing-bound workloads (long system prompts, RAG) — speculation only speeds the *decode* phase.

## Gotchas

1. **`--spec-draft-n-max` is a dial, not a free lunch** — 4–5 is a common sweet spot; 15+ only pays off with DFlash-style block drafts or very high acceptance.
2. **Draft must share the target's tokenizer** — mismatched tokenizers = garbage acceptance. EAGLE-3/DFlash drafts are converted with `--target-model-dir` precisely to inherit it.
3. **MTP needs the model's MTP heads** — `--spec-type draft-mtp` on a model without MTP layers does nothing (or errors); some models ship them as a separate small GGUF (the "MTP draft" files on HF).
4. **n-gram methods are workload-dependent** — brilliant on code, near-useless on novel prose; check the acceptance stats before believing the speedup.
5. **Temp interacts with acceptance** — high `--temp` makes the target's distribution flatter and drafts less likely to match; speculative gains shrink as you crank temperature.
6. **Mixing types is allowed but measure** — `ngram-mod,ngram-map-k4v` can cover more cases, but each adds overhead; benchmark the combination.

## Sources

- [Speculative decoding docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)
- [Server README — spec flags](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [SPEED-Bench client](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/bench/speed-bench/README.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B Q4_K_XL + MTP draft (`--spec-type draft-mtp --spec-draft-n-max 2`): acceptance ~X%, decode Y → Z tok/s (+W%) on chat prompts; ngram-simple on a code-refactor prompt accepted ~V% and added ~U% — llama-bench before/after attached")
