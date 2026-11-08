# Sampling parameters — deep dive

**Summary.** Sampling parameters control how the model picks the next token from its probability distribution over the whole vocabulary. The pipeline is: **logits → (penalties) → truncation filters (`top_k`, `typical_p`, `top_p`, `min_p`) → temperature → renormalize → sample** (or argmax when `temperature=0`). Temperature reshapes the distribution; the truncation filters decide which candidates survive; penalties discourage repetition; `seed` makes runs reproducible. Every runtime applies a *sampler chain* in a fixed order, and the order matters — in llama.cpp the default chain applies penalties first and temperature **last**. One-page version: `cheatsheets/sampling-params.md`; task presets: `cheatsheets/task-settings.md`.

## The pipeline

At every token step the model produces raw scores (logits) for all ~100k+ vocabulary tokens. Sampling transforms them:

```text
logits
  → penalties        (repetition / presence / frequency / DRY — nudge scores of seen tokens)
  → top_n_sigma      (llama.cpp: keep within N standard deviations — off by default)
  → top_k            (keep the K highest, fixed count)
  → typical_p        (keep tokens above a local-entropy threshold)
  → top_p            (keep the smallest set whose probabilities sum to ≥ p)
  → min_p            (keep tokens with P ≥ min_p × P(top token))
  → xtc              (xi-temperature cross-entropy — experimental, off by default)
  → temperature      (divide surviving logits by T)
  → renormalize → sample one token (or argmax if T = 0)
```

Two consequences worth internalizing:

1. **Filters are cumulative** — `top_k=50` + `top_p=0.9` means "no more than 50 candidates, *and* only the 90% nucleus".
2. **Temperature is applied after truncation** (llama.cpp default chain `penalties;dry;top_n_sigma;top_k;typ_p;top_p;min_p;xtc;temperature`) — so `top_p` selects the nucleus from the *un-tempered* distribution, then temperature reshapes whatever survived.

## temperature

Divides the logits by T before the final softmax. Effect on a 4:1 preference between two tokens:

| T | Effective ratio | Feel |
|---|---|---|
| 0 | ∞ (argmax) | greedy, deterministic — always the single most likely token |
| 0.3 | 4^3.3 ≈ 64:1 | near-greedy, very safe |
| 0.7 | ≈ 8:1 | focused, slight variety |
| 1.0 | 4:1 | the model's native distribution |
| 1.5 | ≈ 2.6:1 | flatter, more surprising tokens |
| 2.0 | 2:1 | very loose — expect more nonsense |

- **T = 0 is deterministic** (argmax every step) — the standard choice for reproducible pipelines and JSON/structured output. It is *not* "best possible output": greedy decoding can lock into loops, which is why some models are tuned to behave best at low-but-nonzero T.
- **T > 1** is for creative work (stories, brainstorming). Most models degrade into incoherence past ~1.3–1.5.
- Rule of thumb: **use temperature as your primary knob**, add `top_p` for fine-tuning, treat `top_k` as a safety net.

## top_k

Keeps only the K most likely tokens, discards the rest, renormalizes. Fixed count regardless of how confident the model is.

- `0` (or `-1` in vLLM) = disabled.
- Small K (5–20) = very constrained; large K (100+) ≈ no effect on most models.
- Weakness: when the model is confident, top-k is irrelevant; when it's uncertain, a fixed K keeps garbage. That's why top-p is usually the better primary filter.

## top_p (nucleus sampling)

Keeps the *smallest set* of tokens whose cumulative probability is ≥ p, discards the rest. Adaptive: confident model → tiny nucleus; uncertain model → larger nucleus.

- `1.0` = disabled.
- The standard practical combo from the original paper (Holtzman et al. 2019) is `temperature=0.8, top_p=0.95`.
- Don't stack aggressive `top_k` *and* aggressive `top_p` — they fight over the same candidates.

## min_p

Keeps tokens with probability ≥ `min_p × P(most likely token)` — a *relative* floor. Introduced by Karpathy; especially effective on small models, where the tail of the distribution is noisy.

- `0.0` = disabled.
- `0.05` is the common default (llama.cpp server default).
- Often the single best knob for small local models: it prunes the long tail without the rigidity of a fixed K.

## typical_p

Keeps tokens whose probability exceeds a threshold derived from the local entropy of the distribution (the "typical set"). Niche in practice — most people get the same effect from `min_p` + `top_p`. llama.cpp default `1.0` = off.

## Repetition penalties

Three flavors, all applied to tokens already in the context:

| Penalty | Math | Behavior |
|---|---|---|
| `repetition_penalty` | multiplicative on logits: `logit /= p` if logit > 0, `× p` if logit < 0 (p > 1) | penalizes *any* token seen in the last N (llama.cpp: `--repeat-last-n 64`) |
| `presence_penalty` | `logit -= p` once if the token appears at all | OpenAI-style; discourages *topics* from recurring |
| `frequency_penalty` | `logit -= p × count` per occurrence | OpenAI-style; stronger for tokens used many times |

- `1.0` (or `0.0`) = off — the default everywhere.
- **Use sparingly.** 1.05–1.15 for long generations that start looping; > 1.2 and text starts sounding evasive ("and so on", "etc.", skipped words).
- llama.cpp also has **DRY** (`--dry-multiplier`) — a more aggressive context-aware penalty; leave at 0 unless you know why you need it.
- Reasoning models: penalties applied to *thinking* tokens can corrupt the chain of thought — prefer `stop`/budget controls over penalties there.

## seed

Fixes the RNG so the same prompt + params + model produce the same output.

- `seed` alone is not enough: the model, runtime version, and (for servers) batching can also affect results. For strict reproducibility: same model file, same runtime, single request at a time, fixed seed, `temperature=0`.
- A/B testing: keep `seed` constant while changing one parameter, or the difference you see is noise.
- OpenAI-compatible APIs: `seed` is accepted by llama.cpp, Ollama, and vLLM; vLLM's continuous batching can still break bit-exact reproducibility across runs.

## Per-runtime defaults

| Param | llama.cpp server | Ollama | vLLM |
|---|---|---|---|
| temperature | 0.80 | 0.8 | 1.0 |
| top_k | 40 (0 = off) | 40 | -1 (off) |
| top_p | 0.95 | 0.9 | 1.0 (off) |
| min_p | 0.05 | 0.0 (off) | 0.0 (off) |
| typical_p | 1.0 (off) | — | — |
| repetition_penalty | 1.0 (off) | 1.0 (off) | 1.0 (off) |
| presence/frequency | 0.0 | — | 0.0 |
| seed | -1 (random) | 0 (random) | None (random) |

vLLM's defaults are "raw model" (no shaping at all) — you set everything. llama.cpp/Ollama ship opinionated chat defaults.

## Model-recommended settings

Model cards often specify recommended sampling — follow them, they're tuned per model:

- **Qwen3 (and Qwen3.8)**: thinking mode `temperature=0.6, top_p=0.95, top_k=20, min_p=0`; non-thinking `temperature=1.0, top_p=0.95, top_k=20`. (This is the setup in `testcommands` at the repo root.)
- **Llama 3.x**: `temperature=0.6, top_p=0.9, top_k=40, min_p=0.05` (Meta's recommended).
- **Mistral**: `temperature=0.7, top_p=0.9` (their default).
- **DeepSeek-R1**: `temperature=0.6, top_p=0.95, top_k=20` (no penalties).

## Common mistakes

1. **Cranking everything** — `temp=0.2, top_k=5, top_p=0.1` doesn't make the model "more careful"; it makes it degenerate. Change one knob at a time.
2. **Penalties as a quality lever** — a 1.3 repetition penalty to "fix" repetition usually means the prompt or the model is the problem; the penalty just masks it badly.
3. **Forgetting the model's recommended range** — a model tuned for `temp=0.6` will sound off at `temp=1.2` no matter what the UI default says.
4. **A/B testing without a fixed seed** — you're comparing noise.
5. **Assuming `temp=0` + `top_p` still samples** — at T=0 the output is argmax; the filters only matter when T > 0 (except that they can still change *which* token is argmax in some implementations — know your runtime).

## Sources

- [HF — Generation parameters (temperature, top_p, top_k, min_p, penalties)](https://huggingface.co/docs/transformers/en/generation_strategies)
- [Nucleus sampling paper (Holtzman et al. 2019)](https://arxiv.org/abs/1904.09751)
- [CTRL paper — repetition penalty (Keskar et al. 2019)](https://arxiv.org/abs/1909.05858)
- [llama.cpp — samplers (default chain, DRY, xtc, min_p)](https://github.com/ggml-org/llama.cpp/blob/master/common/sampling.cpp)
- [llama.cpp server README — sampling flags](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Ollama — Modelfile parameters](https://docs.ollama.com/modelfile)
- [vLLM — SamplingParams API](https://docs.vllm.ai/en/stable/api/vllm/sampling_params/)
- [Qwen3 usage guide — recommended sampling](https://qwen.readthedocs.io/en/latest/usage/sampling_params.html)
- [Karpathy — min_p (introduction)](https://x.com/karpathy/status/1922636992695723328)

## My experiments

- (fill in: e.g. "Qwen3.8-27B on :8080 with card defaults (0.6/0.95/20/0) vs llama.cpp defaults (0.8/0.95/40/0.05): card defaults measurably tighter on code; `repetition_penalty 1.1` fixed a looping failure mode on long RAG answers but made short answers evasive; `temp=0` + `top_p=0.95` gave stable JSON with zero schema violations across 50 runs, seed fixed")
