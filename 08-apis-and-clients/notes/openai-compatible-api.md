# The OpenAI-compatible API — the common ground

**Summary.** The OpenAI API shape (`POST /v1/chat/completions` + friends) became the de-facto standard for local inference: **Ollama, llama.cpp, and vLLM all speak it**, so one client works against any runtime — you only change `base_url` and the model name. This note is the cross-runtime view: the endpoints that work everywhere, the request/response skeleton, per-runtime support differences, and the gotchas that bite when you switch runtimes. Runtime-specific deep dives: [03-ollama/notes/rest-api.md](../../03-ollama/notes/rest-api.md), [04-llama-cpp/notes/api.md](../../04-llama-cpp/notes/api.md), [05-vllm](../../05-vllm/). One-page version: `cheatsheets/openai-api.md`.

## Why this shape won

- **One client, any backend** — point the `openai` SDK (or any OpenAI-compatible client) at `http://localhost:PORT/v1` and it works. Swapping Ollama → llama.cpp → vLLM is a config change, not a code change.
- **The ecosystem assumes it** — frontends (Open WebUI, LibreChat, AnythingLLM), agent frameworks, and dev tools all target this API first.
- **It's a moving target, but a compatible one** — new fields (`reasoning_effort`, `response_format` schemas, the Responses API) get added; old ones keep working.

## Base URL & auth per runtime

| Runtime | `base_url` | Auth | Model name to use |
|---|---|---|---|
| Ollama | `http://localhost:11434/v1` | none (or `OLLAMA_API_KEY`) | library tag, e.g. `qwen3:8b` |
| llama.cpp | `http://localhost:8080/v1` | none unless `--api-key` | `--alias` (else GGUF filename) |
| vLLM | `http://localhost:8000/v1` | none unless `--api-key` | served model id (e.g. `Qwen/Qwen3-8B`) |

All three also accept a dummy `api_key="none"` in SDKs that require one.

## The endpoint map (what works where)

| Endpoint | Ollama | llama.cpp | vLLM | Notes |
|---|---|---|---|---|
| `GET /v1/models` | ✅ | ✅ | ✅ | llama.cpp adds `meta` (params, ctx); vLLM adds `owned_by` |
| `POST /v1/chat/completions` | ✅ | ✅ | ✅ | the workhorse; streaming SSE everywhere |
| `POST /v1/completions` | ✅ | ✅ | ✅ | legacy prompt API — prefer chat for instruct models |
| `POST /v1/embeddings` | ✅ | ✅ (`--embedding`) | ✅ | embedding models only |
| `POST /v1/responses` | — | ✅ | ✅ | newer; tool loops, `reasoning_effort` |
| `POST /v1/messages` (Anthropic) | — | ✅ | — | Claude-SDK compat layer (llama.cpp only) |
| `GET /health` | `/api/version` | ✅ | ✅ | health-check shape differs |

## The request/response skeleton

```bash
curl -s http://localhost:8080/v1/chat/completions -d '{
  "model": "qwen3.8",
  "messages": [
    {"role": "system", "content": "You are terse."},
    {"role": "user", "content": "Explain KV cache in one line."}
  ],
  "stream": false,
  "temperature": 0.6,
  "max_tokens": 256
}'
```

```json
{
  "id": "chatcmpl-…",
  "model": "qwen3.8",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "…"},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 42, "completion_tokens": 37, "total_tokens": 79}
}
```

- `finish_reason`: `stop` (model ended the turn) vs `length` (hit `max_tokens`) vs `tool_calls` — **always check it** in scripts.
- `usage` is present on all three runtimes; llama.cpp additionally returns a `timings` object (tok/s) — the easiest built-in perf signal.
- **Streaming** (`"stream": true`): Server-Sent Events — lines of `data: {…}` with `choices[0].delta`, ending in `data: [DONE]`. Same format on all three runtimes.

## Feature support matrix

| Feature | Ollama | llama.cpp | vLLM |
|---|---|---|---|
| Sampling (`temperature`, `top_p`, `top_k`, `min_p`, `seed`) | ✅ (`options`) | ✅ | ✅ |
| JSON mode (`response_format: json_object`) | ✅ (`format: json` native) | ✅ | ✅ |
| JSON **schema** constraint | partial | ✅ (`json_schema`) | ✅ (`guided_json`) |
| Tool calling (`tools`, `tool_choice`) | ✅ | ✅ | ✅ |
| Reasoning fields (`reasoning_content`) | `think` param | ✅ | ✅ (`--reasoning-parser`) |
| Images (`image_url` content parts) | ✅ | ✅ | ✅ |
| `logprobs` | — | ✅ | ✅ |
| `n` (parallel candidates) | — | partial | ✅ |

Rule of thumb: **chat + sampling + JSON + tools + images work everywhere** — build on that common core, and treat the rest as runtime-specific.

## Cross-runtime gotchas

1. **Model name is per-runtime** — `qwen3:8b` (Ollama tag) vs `qwen3.8` (llama.cpp `--alias`) vs `Qwen/Qwen3-8B` (vLLM served id). Keep it in config, never hardcode.
2. **llama.cpp `model` must match `--alias`** — wrong name → 404 even though the model is loaded.
3. **`max_tokens` vs `max_completion_tokens`** — older runtimes read `max_tokens`; newer OpenAI uses `max_completion_tokens`. All three locals accept `max_tokens`; check before sending the other.
4. **Reasoning content splits** — with thinking on, `content` may be empty while `reasoning_content` streams (llama.cpp/vLLM); Ollama uses its own `think` param + `thinking` field. Clients must read both.
5. **Ollama's native API is not OpenAI-shaped** — `/api/chat` returns `message.content`, not `choices[0].message.content`. Don't mix the two dialects in one codebase.
6. **Context limits differ** — a 32k prompt gets a 400/413 on a 4k-context server; read the error, check the runtime's ctx setting.
7. **Auth is opt-in** — no key by default on all three; if you expose on LAN, set one (see [10-deployment](../10-deployment/)).

## Sources

- [OpenAI API reference (the canonical shape)](https://platform.openai.com/docs/api-reference)
- [Ollama — OpenAI compatibility](https://docs.ollama.com/api)
- [llama.cpp server README — API endpoints](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [vLLM — OpenAI compatibility](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/)
- [This repo — 04-llama-cpp/notes/api.md (llama.cpp deep dive)](../../04-llama-cpp/notes/api.md)
- [This repo — 03-ollama/notes/rest-api.md (Ollama deep dive)](../../03-ollama/notes/rest-api.md)

## My experiments

- (fill in: e.g. "same Python script against all three runtimes: only `base_url` + model name changed; llama.cpp's `timings` object was the only per-runtime extra I kept; Ollama dropped `logprobs` silently, vLLM rejected nothing — support matrix above matches what I hit")
