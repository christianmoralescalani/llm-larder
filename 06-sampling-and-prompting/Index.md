# Index — 06-sampling-and-prompting

**How to read this folder.** This is the *quality & speed dial* folder: what controls how the model answers (sampling), how thinking models work, how the prompt is assembled, and how to measure performance. Read in order — sampling first, because the other notes reference it.

## Start here

`notes/sampling-parameters.md` — the knobs (`temperature`, `top_p`, `top_k`, `min_p`, …) and what each one actually does.

## Reading order

1. `notes/sampling-parameters.md` — every knob, recommended ranges, what breaks
2. `notes/reasoning-models.md` — thinking models: budgets, `reasoning_content`, parsing *(you're running one — Qwen3.8)*
3. `notes/chat-templates.md` — how the prompt is assembled, `--jinja`, what a wrong template does
4. `notes/performance-metrics.md` — TTFT, tokens/sec, prompt processing — and how to measure them
5. `notes/prompting-basics.md` — system prompts, few-shot, structured output for local models

## Quick reference (cheatsheets)

- `cheatsheets/sampling-params.md` — the knobs + ranges, one page
- `cheatsheets/reasoning-controls.md` — thinking on/off/budget per runtime
- `cheatsheets/perf-metrics.md` — what to measure, good numbers, how

## Sources & videos

- `docs/references.md` — official docs
- `videos.md` — curated videos
