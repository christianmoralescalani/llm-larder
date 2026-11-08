# REST API

**Summary.** Ollama speaks plain HTTP on port 11434: NDJSON streaming by default, single JSON with `"stream": false`. The core endpoints: `POST /api/generate` (prompt → text), `POST /api/chat` (messages → message), `GET /api/tags` (local models), `GET /api/ps` (loaded models), plus management (`/api/pull`, `/api/create`, …) and an OpenAI-compatible `/v1/*` layer. From the host, everything is `curl http://localhost:11434/…`.

## Base URL

| From | URL |
|---|---|
| host (port mapped) | `http://localhost:11434` |
| another container (same compose network) | `http://ollama:11434` |
| LAN | `http://<host-ip>:11434` |

## Version

```bash
curl http://localhost:11434/api/version
# {"version":"0.40.14"}
```

## List models — `GET /api/tags`

```bash
curl http://localhost:11434/api/tags
```

```json
{
  "models": [
    {
      "name": "llama3.2:latest",
      "modified_at": "2026-11-03T10:00:00Z",
      "size": 2026583616,
      "digest": "501a7d391971…",
      "details": {
        "format": "gguf", "family": "llama", "families": ["llama"],
        "parameter_size": "3.2B", "quantization_level": "Q4_K_M"
      }
    }
  ]
}
```

## List loaded models — `GET /api/ps`

```bash
curl http://localhost:11434/api/ps
```

```json
{
  "models": [
    {
      "name": "llama3.2:latest",
      "size": 2026583616,
      "details": { "format": "gguf", "family": "llama", "parameter_size": "3.2B", "quantization_level": "Q4_K_M" },
      "expires_at": "2026-11-03T10:05:00Z",
      "size_vram": 1720000000,
      "context_length": 4096
    }
  ]
}
```

`size_vram` = bytes in VRAM; `expires_at` = when it auto-unloads (keep-alive).

## Generate — `POST /api/generate`

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

Response (non-streaming):

```json
{
  "model": "llama3.2",
  "created_at": "2026-11-03T10:00:00Z",
  "response": "The sky is blue because …",
  "done": true,
  "done_reason": "stop",
  "total_duration": 174560334,
  "load_duration": 101397084,
  "prompt_eval_count": 11,
  "prompt_eval_duration": 13074791,
  "eval_count": 18,
  "eval_duration": 52479709
}
```

**Speed math:** `eval_count / (eval_duration / 1e9)` = decode tok/s; `prompt_eval_count / (prompt_eval_duration / 1e9)` = prefill tok/s. (Durations are nanoseconds.)

Request fields:

| Field | Notes |
|---|---|
| `model` | required |
| `prompt` | text |
| `system` | system prompt |
| `suffix` | for fill-in-the-middle |
| `images` | base64 array (vision models) |
| `format` | `"json"` or a JSON schema → structured output |
| `stream` | default `true` (NDJSON); `false` = single JSON |
| `think` | `true`/`false`/`null` or a model-defined level (e.g. `"low"`) — reasoning models |
| `raw` | skip template processing |
| `keep_alive` | `"10m"`, `3600`, `-1` (keep loaded), `0` (unload after) |
| `options` | `temperature`, `top_k`, `top_p`, `min_p`, `seed`, `stop`, `num_ctx`, `num_predict`, … |
| `logprobs` / `top_logprobs` | token log-probabilities |

### Streaming

```bash
curl -N http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Count to five."
}'
# one JSON object per line (NDJSON); the last has "done": true
```

### Structured output

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Populations of the US and Canada",
  "stream": false,
  "format": {
    "type": "object",
    "properties": {
      "countries": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "country": {"type": "string"},
            "population": {"type": "integer"}
          },
          "required": ["country", "population"]
        }
      }
    },
    "required": ["countries"]
  }
}'
```

### Preload / unload

```bash
curl http://localhost:11434/api/generate -d '{"model": "llama3.2", "keep_alive": -1}'   # load & keep
curl http://localhost:11434/api/generate -d '{"model": "llama3.2", "keep_alive": 0}'    # unload after
```

## Chat — `POST /api/chat`

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [
    {"role": "system", "content": "You are terse."},
    {"role": "user", "content": "why is the sky blue?"}
  ],
  "stream": false
}'
```

Response:

```json
{
  "model": "llama3.2",
  "message": { "role": "assistant", "content": "Rayleigh scattering." },
  "done": true,
  "done_reason": "stop",
  "prompt_eval_count": 24,
  "eval_count": 5,
  "total_duration": 900000000,
  "eval_duration": 400000000
}
```

Extra request fields: `tools` (function calling — results land in `message.tool_calls`), `images` per user message (vision), `think`, `format`, `options`, `keep_alive`, `logprobs`.

## Management endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/show` | POST | model details (Modelfile, parameters, template) |
| `/api/create` | POST | build a model (Modelfile in the body) |
| `/api/copy` | POST | rename/tag |
| `/api/pull` | POST | download (streamed progress) |
| `/api/push` | POST | upload |
| `/api/delete` | POST | delete (`{"model": "name"}`) |
| `/api/embed` | POST | embeddings (`{"model": "nomic-embed-text", "input": "…"}`) |

```bash
curl http://localhost:11434/api/delete -d '{"model": "llama3.2:latest"}'
curl http://localhost:11434/api/embed -d '{"model": "nomic-embed-text", "input": "hello world"}'
```

## OpenAI-compatible layer

Ollama also serves `/v1/chat/completions`, `/v1/completions`, `/v1/models` (and an Anthropic-compatible `/v1/messages`) — so any OpenAI client works with `base_url=http://localhost:11434/v1`. Full treatment in [08-apis-and-clients](../../08-apis-and-clients/).

```bash
curl http://localhost:11434/v1/chat/completions -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "hi"}]
}'
```

## Sources

- [API — generate](https://docs.ollama.com/api/generate) · [chat](https://docs.ollama.com/api/chat) · [tags](https://docs.ollama.com/api/tags) · [ps](https://docs.ollama.com/api/ps)
- [OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- [OpenAPI spec](https://docs.ollama.com/openapi.yaml)

## My experiments

- (fill in: e.g. "stream:false + jq on /api/generate: 27B Q4 decode at 38 tok/s, prefill 410 tok/s on my 24 GB card")
