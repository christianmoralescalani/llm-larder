# OpenAI-compatible API (and friends)

**Summary.** `llama-server` speaks three dialects: **OpenAI-compatible** (`/v1/chat/completions`, `/v1/completions`, `/v1/models`, `/v1/embeddings`, `/v1/responses`), **Anthropic-compatible** (`/v1/messages`), and **native llama.cpp** endpoints (`/completion`, `/embedding`, `/tokenize`, `/slots`, `/metrics`, …). Any OpenAI SDK works by pointing `base_url` at `http://localhost:8080/v1` with a dummy key. Streaming is SSE for `/v1/*` and NDJSON-ish chunks for native endpoints; both default to streaming.

## Point an OpenAI client at it

```python
import openai
client = openai.OpenAI(base_url="http://localhost:8080/v1", api_key="sk-no-key-required")
r = client.chat.completions.create(
    model="qwen3.8",  # must match --alias (or the GGUF name)
    messages=[{"role": "user", "content": "hello"}],
    stream=False,
)
print(r.choices[0].message.content)
```

```bash
# no SDK needed
curl -s http://localhost:8080/v1/chat/completions -d '{
  "model": "qwen3.8",
  "messages": [{"role": "user", "content": "Explain KV cache"}],
  "stream": false
}'
```

Auth: if the server was started with `--api-key KEY`, send `Authorization: Bearer KEY` (or `api_key=KEY` in the SDK).

## Endpoint map

| Endpoint | Dialect | Purpose |
|---|---|---|
| `GET /health` (also `/v1/health`) | — | 200 `{"status":"ok"}` or 503 while loading (no auth) |
| `GET /v1/models` | OpenAI | loaded model info + `meta` (params, size, ctx) + modalities |
| `POST /v1/chat/completions` | OpenAI | chat (tools, reasoning, response_format, streaming) |
| `POST /v1/completions` | OpenAI | legacy prompt completions |
| `POST /v1/embeddings` | OpenAI | embeddings (run server with `--embedding` for embedding models) |
| `POST /v1/responses` | OpenAI | Responses API (incl. tool loops) |
| `POST /v1/messages` | Anthropic | Messages API (Claude SDKs) |
| `POST /completion` | native | prompt → completion (not OAI-shaped) |
| `POST /embedding` | native | embeddings (multimodal-capable) |
| `POST /tokenize` / `/detokenize` | native | text ↔ token ids (`with_pieces` for pieces) |
| `POST /apply-template` | native | render the chat template without inference (debugging!) |
| `POST /reranking` | native | rerank docs for a query (server needs `--rerank`) |
| `POST /tts` | native | speech synthesis (TTS models) |
| `GET /slots` | native | per-slot state: running request, tokens, cache reuse |
| `GET /metrics` | native | Prometheus (server needs `--metrics`) |
| `GET/POST /props` | native | read/change global properties (`--props`) |
| `GET/POST /lora-adapters` | native | list/switch LoRA adapters at runtime |
| `GET /models`, `POST /models/load`, `POST /models/unload` | router | multi-model router mode |
| `WS /v1/realtime` | OpenAI | realtime transcription WebSocket |

## `/v1/chat/completions` — the important options

Standard OpenAI fields work: `model`, `messages`, `stream`, `temperature`, `top_p`, `top_k`, `min_p`, `max_tokens`, `stop`, `seed`, `presence_penalty`, `frequency_penalty`, `n`, `logprobs`, `user`. llama.cpp extras:

| Field | Notes |
|---|---|
| `response_format` | `{"type":"json_object"}` or `{"type":"json_schema","schema":{...}}` — schema-constrained output |
| `tools` / `tool_choice` / `parallel_tool_calls` | function calling; native tool-call style when the template supports it, generic style as fallback |
| `reasoning_effort` | `none` disables thinking; otherwise passed to the Jinja template |
| `reasoning_format` | `none` = raw text; else thoughts land in `message.reasoning_content` (DeepSeek-style) |
| `reasoning_control` | arm realtime control so the completion can be ended via `/v1/chat/completions/control` |
| `cache_prompt` | per-request prompt-cache toggle (server default `--cache-prompt`) |
| `mirostat`, `dry_*`, `xtc_*`, `logit_bias` | native sampler knobs also accepted |

Non-streaming responses include a `timings` object (prompt/eval token counts and durations → tok/s) and a standard `usage` object.

```bash
# structured output
curl -s http://localhost:8080/v1/chat/completions -d '{
  "model": "qwen3.8",
  "messages": [{"role":"user","content":"Population of France"}],
  "response_format": {"type":"json_object"},
  "stream": false
}'

# streaming: SSE lines `data: {…}` ending with `data: [DONE]`
curl -N http://localhost:8080/v1/chat/completions -d '{
  "model": "qwen3.8",
  "messages": [{"role":"user","content":"count to 5"}],
  "stream": true
}'
```

## Native endpoints worth knowing

```bash
# what tokens did the model see?
curl -s http://localhost:8080/tokenize -d '{"content": "hello world", "with_pieces": true}'

# render the chat template (no inference) — the #1 template-debugging tool
curl -s http://localhost:8080/apply-template -d '{
  "messages": [{"role":"system","content":"You are terse."},
               {"role":"user","content":"hi"}]
}'

# who is running where, and what cache is being reused
curl -s http://localhost:8080/slots | python3 -m json.tool

# prometheus
curl -s http://localhost:8080/metrics | grep llama
```

## Gotchas

1. **`model` must match `--alias`** (or the GGUF filename) — wrong name → 404, even though the model is loaded.
2. **Streaming format differs per dialect** — `/v1/*` is OpenAI SSE (`data: …` + `[DONE]`); native `/completion` streams its own chunk shape. Don't mix parsers.
3. **`/v1/chat/completions` needs a chat template** — models without a supported template fall back to ChatML and may misbehave; check with `/apply-template` or `llama-cli --verbose-prompt`.
4. **`--embedding` mode is restrictive** — an embedding-model server only serves embedding endpoints; don't expect chat from it.
5. **`/metrics` and `/reranking` are off by default** — `--metrics`, `--rerank`.
6. **`/v1/models` `id` is the file path** unless you set `--alias`; scripts that hardcode OpenAI model names need the alias.
7. **Anthropic `/v1/messages`** is a compatibility layer for Claude SDKs — same model, Anthropic request/response shape; don't send Anthropic model names.
8. **Reasoning content** — with `--reasoning on` (or `auto` + a thinking template), thoughts appear in `reasoning_content`, not `content`; clients that only read `content` will see an empty-looking answer until thinking ends.

## Sources

- [Server README — API Endpoints](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Function calling docs](https://github.com/ggml-org/llama.cpp/blob/master/docs/function-calling.md)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference)

## My experiments

- (fill in: e.g. "OpenAI python SDK against :server-cuda + Qwen3.8 — `base_url …/v1`, alias `qwen3.8`: chat, streaming, `response_format json_schema`, and tool calls all work; `timings` shows prompt 612 t/s / eval 38 t/s; `/apply-template` confirmed the Jinja template renders the system prompt correctly; Anthropic SDK via /v1/messages also answered")
