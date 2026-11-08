# Cheatsheet — reasoning (thinking) controls

**TL;DR.** Thinking models emit `<thinking>…</thinking>` before the answer; thinking tokens are *generated* tokens (they cost context, latency, and `max_tokens` budget). Controls: **on/off** (template `enable_thinking`), **effort** (`minimal`…`max`), **token budget**. Runtimes parse the block into a separate field — llama.cpp `message.reasoning_content`, Ollama `message.thinking`, vLLM `reasoning_content` — so never regex the answer. Deep dive: `notes/reasoning-models.md`.

## Per-runtime controls

| Runtime | On | Off | Effort / budget | Where thinking lands |
|---|---|---|---|---|
| **llama.cpp** | `--reasoning on` (default `auto`) | `--reasoning off` | `--reasoning-effort minimal…max`, `--reasoning-budget N` | `message.reasoning_content` (with `--reasoning-format deepseek`) |
| **Ollama CLI** | `ollama run <m> --think` | `--think=false` (or `/set nothink` in session) | `--think high/medium/low` | — (hidden in UI) |
| **Ollama API** | `"think": true` (top-level) | `"think": false` | `"think": "high"|"medium"|"low"` | `message.thinking` (separate from `content`) |
| **vLLM** | default for Qwen3/R1 | `--default-chat-template-kwargs '{"enable_thinking": false}'` (server) or per-request `chat_template_kwargs` | `reasoning_effort` (Responses API); parser: `--reasoning-parser qwen3\|deepseek_r1\|…` | `reasoning_content` (needs `--reasoning-parser`); `include_reasoning: false` to hide |

## Quick recipes

```bash
# llama.cpp: thinking on, 4k budget, deepseek-style parsing
--reasoning on --reasoning-format deepseek --reasoning-budget 4096

# Ollama: think at medium effort
curl http://localhost:11434/api/chat -d '{"model":"qwen3:8b","think":"medium",
  "messages":[{"role":"user","content":"..."}]}'

# vLLM: serve Qwen3 with thinking parsed out
vllm serve Qwen/Qwen3-8B --reasoning-parser qwen3
# vLLM: server-wide non-thinking
vllm serve Qwen/Qwen3-8B --reasoning-parser qwen3 \
  --default-chat-template-kwargs '{"enable_thinking": false}'
```

## Sampling for thinking mode (Qwen3 example)

| Mode | temperature | top_p | top_k | min_p |
|---|---|---|---|---|
| thinking on | 0.6 | 0.95 | 20 | 0 |
| thinking off | 1.0 | 0.95 | 20 | 0 |

## Decision table

| Situation | Do this |
|---|---|
| Math / logic / hard debugging | thinking **on**, effort high, budget generous |
| Chit-chat / lookup / formatting | thinking **off** (latency + token waste) |
| JSON / structured pipeline | thinking **off**, or on + reasoning parser so `content` stays clean |
| Interactive use, latency matters | thinking on + **budget cap** (e.g. 2048) or effort `low` |
| Batch / overnight jobs | thinking on, effort `max`, no budget |

## Gotchas

1. **`max_tokens` includes thinking** — cap 512 on a thinking model ≈ 500 thought + 12 answer.
2. **`think: false` isn't universal** — some Qwen3-MoE builds ignored it (Ollama #17292/#17588, mid-2026); verify per model.
3. **Never apply repetition penalties to thinking tokens** — they corrupt the chain of thought.
4. **Truncated thinking = broken answer** — `finish_reason: "length"` mid-thought → empty/partial content.
5. **Stale GGUF template = no toggle** — the on/off switch lives in the template; old quants may always-think.
6. **History**: Qwen3-style templates strip previous turns' thinking by default (`--reasoning-preserve` in llama.cpp controls it) — that's correct, don't "fix" it.

Verified against llama.cpp master (v0.8.0), Ollama 0.32.x, vLLM 0.9.x — 2026-11-08.
