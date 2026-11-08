# Cheatsheet — sampling parameters

**TL;DR.** `temperature` reshapes the distribution (0 = greedy/deterministic, 1 = native, >1 = wilder); `top_k` keeps the K best tokens; `top_p` keeps the smallest set summing to p (adaptive — usually the better primary filter); `min_p` keeps tokens ≥ `min_p × P(top)` (best single knob for small models); penalties (`repetition/presence/frequency`) nudge against repeats (use ≤1.15 or not at all); `seed` = reproducible. Start from the **model's recommended values**, not the runtime default. Deep dive: `notes/sampling-parameters.md`.

## The parameters

| Param | What it does | llama.cpp | Ollama | vLLM | Recommended range |
|---|---|---|---|---|---|
| `temperature` | divides logits by T; sharpens (↓) / flattens (↑) | 0.80 | 0.8 | 1.0 | 0 (deterministic) · 0.3–0.7 (focused/code) · 0.7–1.0 (chat) · 1.0–1.3 (creative) |
| `top_k` | keep only K most likely tokens | 40 (0=off) | 40 | -1 (off) | 20–40; off for large models |
| `top_p` | nucleus: smallest set with cumulative P ≥ p | 0.95 | 0.9 | 1.0 (off) | 0.85–0.95; 1.0 = off |
| `min_p` | keep tokens with P ≥ min_p × P(top) | 0.05 | 0.0 (off) | 0.0 (off) | 0.05 (small models); 0 = off |
| `typical_p` | keep tokens above local-entropy threshold | 1.0 (off) | — | — | leave off (niche) |
| `repetition_penalty` | multiplicative penalty on tokens in last N (64) | 1.0 (off) | 1.0 (off) | 1.0 (off) | off → 1.05–1.15 max |
| `presence_penalty` | additive, once per distinct token (OpenAI-style) | 0.0 | — | 0.0 | 0–0.5 |
| `frequency_penalty` | additive, per occurrence (OpenAI-style) | 0.0 | — | 0.0 | 0–0.5 |
| `seed` | RNG seed → reproducible output | -1 (random) | 0 (random) | None (random) | fix it when A/B testing |
| `max_tokens` | output cap (incl. thinking tokens!) | -1 (∞) | -1 | None | always cap batch jobs |
| `stop` | stop sequences | — | — | — | model's end-of-turn tokens |

Sampler order (llama.cpp default chain — order matters):
`penalties → dry → top_n_sigma → top_k → typ_p → top_p → min_p → xtc → temperature`

## Task presets

| Task | temperature | top_p | top_k | min_p | penalties |
|---|---|---|---|---|---|
| Chat / general | 0.6–0.8 | 0.9–0.95 | 20–40 | 0.05 | off |
| Coding | 0.2–0.5 | 0.9 | 20 | 0 | off |
| JSON / structured | 0 (greedy) | 1.0 | off | 0 | off + schema constraint |
| RAG / extraction | 0.2–0.5 | 0.9 | 20–40 | 0.05 | off |
| Creative writing | 0.9–1.2 | 0.95 | 40–100 | 0.05 | off |
| Summarization | 0.3–0.6 | 0.9 | 20–40 | 0.05 | off |
| Translation | 0.2–0.5 | 0.9 | 20 | 0 | off |
| Reasoning (thinking on) | 0.6 | 0.95 | 20 | 0 | **never** |

## Model-recommended (follow the card)

| Model | Settings |
|---|---|
| Qwen3 / Qwen3.8 | thinking: `temp 0.6, top_p 0.95, top_k 20, min_p 0` · non-thinking: `temp 1.0, top_p 0.95, top_k 20` |
| Llama 3.x | `temp 0.6, top_p 0.9, top_k 40, min_p 0.05` |
| Mistral | `temp 0.7, top_p 0.9` |
| DeepSeek-R1 | `temp 0.6, top_p 0.95, top_k 20` |

## Setting them per runtime

```bash
# llama.cpp server (startup defaults; per-request overrides via API)
--temp 0.6 --top-p 0.95 --top-k 20 --min-p 0.0 --repeat-penalty 1.0 --seed 42

# Ollama (Modelfile)
PARAMETER temperature 0.6
PARAMETER top_p 0.95
PARAMETER top_k 20
PARAMETER min_p 0.0
PARAMETER seed 42

# Ollama (API, per request)
curl http://localhost:11434/api/chat -d '{
  "model": "qwen3:8b",
  "options": {"temperature": 0.6, "top_p": 0.95, "top_k": 20, "min_p": 0.0, "seed": 42},
  "messages": [{"role": "user", "content": "hi"}]
}'

# vLLM (API, per request)
"temperature": 0.6, "top_p": 0.95, "top_k": 20, "min_p": 0.0, "seed": 42
```

## Quick decisions

- **Output should be identical every time?** → `temperature 0` + fixed `seed` (+ same model/runtime).
- **Small model (≤8B) sounding sloppy?** → add `min_p 0.05`, drop `temperature` to 0.5.
- **Long output looping?** → `repetition_penalty 1.1` (band-aid) — real fix is prompt/context.
- **Creative mode** → `temperature 1.1`, `top_p 0.95`, `top_k 100`.
- **A/B testing?** → fix `seed`, change one knob, compare ≥10 prompts.

Verified against llama.cpp master (v0.8.0), Ollama 0.32.x, vLLM 0.9.x — 2026-11-08.
