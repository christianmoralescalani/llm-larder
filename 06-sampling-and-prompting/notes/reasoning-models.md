# Reasoning models — thinking, budgets, and parsing

**Summary.** Reasoning models (DeepSeek-R1, Qwen3/Qwen3.8, Gemini thinking, GPT-o…) emit a **thinking block** (`<thinking>…</thinking>`) before the final answer — test-time compute traded for accuracy on math, code, and multi-step tasks. The thinking block is controlled by the **chat template** (an `enable_thinking` flag / budget variable), and each runtime parses it out of the raw token stream into a separate API field (`reasoning_content` / `thinking`). The practical controls everywhere: **on/off**, **effort level**, **token budget**, and **how much thinking to keep in history**. Thinking tokens are *generated* tokens — they cost context, latency, and (for hosted APIs) money exactly like answer tokens.

## What "thinking" actually is

- The model was trained (RL with verifiable rewards) to write a chain of thought before answering. The block is **real generated text** — it consumes `max_tokens`, fills the KV cache, and takes the same tokens/sec as everything else.
- Thinking is **on by default** for Qwen3-series and DeepSeek-R1; **off by default** for some (e.g. Granite 3.2, DeepSeek-V3.1) — check the model card.
- Simple questions don't need thinking: a 27B model burning 2k thinking tokens to answer "what's 2+2" is pure waste. Toggle it per task.

## Controls per runtime

### llama.cpp (server + CLI)

| Flag | Values | What it does |
|---|---|---|
| `--reasoning` | `on` / `off` / `auto` (default) | `auto` = detect from the template; force `on`/`off` to override |
| `--reasoning-format` | `none` / `deepseek` / `deepseek-legacy` | where thoughts land in the API response: `deepseek` → `message.reasoning_content` (separate field); `none` → left in `content` |
| `--reasoning-effort` | `minimal` … `max` | maps onto template effort variables (e.g. Qwen3-style effort levels) |
| `--reasoning-budget` | N tokens, -1 = unlimited | thinking token budget — the main cost dial |
| `--reasoning-preserve` | `template` (default) / on / off | whether previous turns' thinking stays in the conversation history sent back to the model |

Per-request, the OpenAI-compatible API accepts a `reasoning` object (enabled/format/effort/budget) that overrides the server defaults. The response message carries `reasoning_content` (with `--reasoning-format deepseek`) — stream it or drop it, your choice.

```bash
# server: reasoning on, deepseek-style parsing, 4k thinking budget
docker run -d --name llm -p 8080:8080 --gpus all \
  -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --host 0.0.0.0 --port 8080 \
  --n-gpu-layers -1 --jinja \
  --reasoning on --reasoning-format deepseek --reasoning-budget 4096
```

### Ollama

- **CLI**: `ollama run qwen3:8b --think` (on) / `--think=false` (off); interactive: `/set think` / `/set nothink`; `--hidethinking` = think but only show the answer (scripting).
- **API**: top-level `think` parameter on `/api/chat` and `/api/generate` — boolean, or a level string (`"low"`, `"medium"`, `"high"`). Thinking content comes back in a **separate field** from `content` (the `thinking` field on the message), so clients never have to regex the answer.
- Works with architectures that expose `enable_thinking` (qwen3, deepseek, glm4…).
- **Gotcha (known bugs, mid-2026)**: `think: false` was reported to be ignored by some Qwen3-MoE builds (issues #17292, #17588) — if a model "always thinks", verify with a fresh pull or fall back to `--hidethinking` / llama.cpp.

```bash
# API: thinking on at medium effort
curl http://localhost:11434/api/chat -d '{
  "model": "qwen3:8b",
  "think": "medium",
  "stream": false,
  "messages": [{"role": "user", "content": "Prove sqrt(2) is irrational."}]
}'
# → message.content = final answer, message.thinking = the chain of thought
```

### vLLM

| Flag / field | Values | What it does |
|---|---|---|
| `--reasoning-parser` | `deepseek_r1`, `qwen3`, `granite4`, `gemma4`, `kimi_k2`, … (25+ built-in) | splits thinking out of the stream into `reasoning_content` on the message (non-OpenAI extension) |
| `--default-chat-template-kwargs` | `'{"enable_thinking": false}'` | server-wide on/off (Qwen3 is on by default; pass `false` to disable) |
| `chat_template_kwargs` (per request, `extra_body`) | `'{"enable_thinking": true}'` | per-request override |
| `reasoning_effort` (Responses API) | `low`/`medium`/`high` | maps to template effort variables |
| `include_reasoning` | `true`/`false` | `false` = thinking still generated (quality unchanged) but excluded from the response — less network traffic |

```bash
vllm serve Qwen/Qwen3-8B --reasoning-parser qwen3
# or server-wide non-thinking:
vllm serve Qwen/Qwen3-8B --reasoning-parser qwen3 \
  --default-chat-template-kwargs '{"enable_thinking": false}'
```

Version note: `enable_thinking=False` + parser was broken before vLLM 0.9.0 (fixed by the dedicated `qwen3` parser).

## Parsing thinking blocks

The raw model output is one token stream: `…<thinking>long chain of thought</thinking>final answer…`. Parsing options, best first:

1. **Runtime parser** (llama.cpp `--reasoning-format deepseek`, Ollama `think`, vLLM `--reasoning-parser`) — you get clean `reasoning_content` + `content` fields, correctly split even mid-stream. **Always prefer this.**
2. **Manual regex** on `content` (`<thinking>(.*?)</thinking>` with DOTALL) — fragile: the model can emit malformed/unclosed tags, and streaming splits tags across chunks. Only as a last resort against an unmanaged runtime.
3. **`--skip-chat-parsing` / `include_reasoning=false`** — deliberately keep/drop the raw stream.

Streaming: with a parser, thinking arrives as separate SSE chunks (llama.cpp: `reasoning_content` in deltas; vLLM: `reasoning_content`/Responses API `reasoning` items). UIs typically render it in a collapsible "thinking…" box and only count `content` as the answer.

## History and context cost

- **Keep or strip?** Qwen3-style templates **strip previous turns' thinking** from history by default (it's no longer needed and costs tokens). `--reasoning-preserve` in llama.cpp controls this; vLLM/Ollama follow the model template's behavior.
- **Budget math**: with a 32k context and a 4k thinking budget, each thinking turn can consume 12.5% of the window. Long multi-turn reasoning conversations will hit context limits far sooner than they look — watch `usage`/`prompt_tokens`.
- **Latency budget**: thinking tokens are decode-bound. A 2k-token thought at 30 tok/s ≈ 67 s before the first answer token. Set `--reasoning-budget` (or effort `low`) for interactive use; reserve `max` for batch jobs.

## When to use thinking

| Task | Thinking? | Why |
|---|---|---|
| Math, logic, multi-step planning | **on** (high effort) | where the accuracy gain is real |
| Hard debugging / code architecture | **on** | catches its own mistakes mid-thought |
| Chit-chat, factual lookup, formatting | **off** | pure latency + token waste |
| JSON/structured pipelines | **off** (or budget-capped) | thinking tokens break naive JSON parsing; parsers handle it, but why pay? |
| RAG with long context | **off/low** | thinking eats the context you paid to fill |

## Gotchas

1. **`max_tokens` includes thinking** — a 512-token cap on a reasoning model often means 500 thought + 12 answer. Budget for it or raise the cap.
2. **Sampling params differ by mode** — Qwen3 recommends T=0.6 while thinking, T=1.0 without; using chat defaults for a thinking model degrades reasoning quality.
3. **Don't penalize thinking tokens** — repetition penalties applied inside the thought block corrupt the chain of thought.
4. **`think: false` isn't universal** — some builds ignore it (Ollama issues above); some models are always-thinking by design. Verify, don't assume.
5. **Truncated thinking = broken answer** — if you hit `max_tokens` mid-`<thinking>`, most parsers return empty/partial content. Check `finish_reason == "length"`.
6. **Old GGUFs, old templates** — the thinking toggle lives in the *template*; a quant built before the model's reasoning support was added won't respect on/off. Rebuild or use a newer quant.

## Sources

- [Ollama — Thinking (blog: CLI + API controls)](https://ollama.com/blog/thinking)
- [Ollama — API reference (`think` parameter)](https://docs.ollama.com/api)
- [vLLM — Reasoning outputs (parsers, `include_reasoning`, template kwargs)](https://docs.vllm.ai/en/latest/features/reasoning_outputs)
- [Qwen docs — vLLM deployment (thinking modes, `reasoning_content`)](https://qwen.readthedocs.io/en/latest/deployment/vllm.html)
- [llama.cpp server README — `--reasoning*` flags](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Google for Developers — How do thinking and reasoning models work?](https://www.youtube.com/watch?v=xCRvOUykOX0)
- [This repo — 04-llama-cpp/notes/llama-server.md (flag reference)](../../04-llama-cpp/notes/llama-server.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B `--reasoning on` vs off on a 20-problem math set: accuracy 61% → 84%, avg latency ×3.2; `--reasoning-budget 2048` kept 90% of the accuracy at half the latency; Ollama `think: \"high\"` on the same model produced the `thinking` field cleanly, no regex needed")
