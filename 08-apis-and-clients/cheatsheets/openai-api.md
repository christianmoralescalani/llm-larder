# Cheatsheet — OpenAI-compatible API (local)

**TL;DR.** All three local runtimes speak the OpenAI API — change only `base_url` + model name. Core: `GET /v1/models`, `POST /v1/chat/completions` (the workhorse, SSE streaming), `POST /v1/embeddings`. Sampling/JSON/tools/images work everywhere. Deep dive: `notes/openai-compatible-api.md`.

## base_url & model name

| Runtime | `base_url` | Auth | Model name |
|---|---|---|---|
| Ollama | `http://localhost:11434/v1` | none (or `OLLAMA_API_KEY`) | `qwen3:8b` (library tag) |
| llama.cpp | `http://localhost:8080/v1` | none unless `--api-key` | `--alias` (else GGUF filename) |
| vLLM | `http://localhost:8000/v1` | none unless `--api-key` | served id (`Qwen/Qwen3-8B`) |

SDKs require a key → pass `api_key="none"`.

## Endpoints

| Endpoint | Ollama | llama.cpp | vLLM |
|---|---|---|---|
| `GET /v1/models` | ✅ | ✅ (+`meta`) | ✅ |
| `POST /v1/chat/completions` | ✅ | ✅ (+`timings`) | ✅ |
| `POST /v1/completions` | ✅ | ✅ | ✅ |
| `POST /v1/embeddings` | ✅ | ✅ (`--embedding`) | ✅ |
| `POST /v1/responses` | — | ✅ | ✅ |
| `POST /v1/messages` (Anthropic) | — | ✅ | — |

## Request skeleton

```json
{
  "model": "qwen3.8",
  "messages": [
    {"role": "system", "content": "You are terse."},
    {"role": "user", "content": "Explain KV cache in one line."}
  ],
  "stream": false,
  "temperature": 0.6,
  "max_tokens": 256
}
```

Response: `choices[0].message.content` · `choices[0].finish_reason` (`stop`/`length`/`tool_calls`) · `usage.{prompt,completion,total}_tokens`. Streaming: `data: {…}` SSE lines with `choices[0].delta`, ending `data: [DONE]`.

## Feature matrix

| Feature | Ollama | llama.cpp | vLLM |
|---|---|---|---|
| Sampling (temp/top_p/top_k/min_p/seed) | ✅ | ✅ | ✅ |
| JSON mode / schema constraint | ✅ / partial | ✅ | ✅ |
| Tools | ✅ | ✅ | ✅ |
| Images (`image_url`) | ✅ | ✅ | ✅ |
| Reasoning fields | `think` param | `reasoning_content` | `reasoning_content` |
| `logprobs` | — | ✅ | ✅ |

## Top gotchas

1. Model name is per-runtime — wrong name → 404.
2. `max_tokens` (portable) vs `max_completion_tokens` (new OpenAI) — use `max_tokens` locally.
3. Thinking models: `content` may be empty while `reasoning_content` streams.
4. Ollama's native `/api/*` is **not** OpenAI-shaped — don't mix dialects.
5. API is stateless — send the full history every request.

Verified against Ollama 0.40.x, llama.cpp master (v0.8.0), vLLM 0.9.x — 2026-11-08.
