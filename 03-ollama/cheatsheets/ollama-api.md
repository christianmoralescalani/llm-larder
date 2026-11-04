# Cheatsheet — Ollama REST API

**TL;DR.** Base `http://localhost:11434` (host) / `http://ollama:11434` (sibling container). Chat: `POST /api/chat` with `messages`; completions: `POST /api/generate` with `prompt`; list: `GET /api/tags`; loaded: `GET /api/ps`. Add `"stream": false` for single JSON (default is NDJSON streaming). tok/s = `eval_count / (eval_duration/1e9)`. Verified on `0.40.14`, 2026-11-03.

## Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/version` | GET | server version |
| `/api/tags` | GET | local models |
| `/api/ps` | GET | loaded models (VRAM, expiry, context) |
| `/api/generate` | POST | prompt → text (images, format, think, keep_alive) |
| `/api/chat` | POST | messages → message (tools, images, think) |
| `/api/show` | POST | model details |
| `/api/create` | POST | build a model |
| `/api/copy` | POST | rename/tag |
| `/api/pull` / `/api/push` | POST | download / upload |
| `/api/delete` | POST | delete a model |
| `/api/embed` | POST | embeddings |
| `/v1/chat/completions`, `/v1/completions`, `/v1/models` | — | OpenAI-compatible |

## Examples

```bash
# chat (single JSON)
curl -s http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "why is the sky blue?"}],
  "stream": false
}'

# streaming (NDJSON, one object per line)
curl -N http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "count to 5"}]
}'

# with sampling options + keep-alive
curl -s http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Explain KV cache",
  "stream": false,
  "options": {"temperature": 0.3, "num_ctx": 8192, "seed": 42},
  "keep_alive": "10m"
}'

# structured output
curl -s http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Population of France",
  "stream": false,
  "format": {"type":"object","properties":{"population":{"type":"integer"}},"required":["population"]}
}'

# vision (base64 image)
IMG=$(base64 < photo.jpg | tr -d '\n')
curl -s http://localhost:11434/api/chat -d '{
  "model": "gemma3",
  "messages": [{"role":"user","content":"Describe","images":["'"$IMG"'"]}],
  "stream": false
}'

# management
curl -s http://localhost:11434/api/tags
curl -s http://localhost:11434/api/ps
curl -s http://localhost:11434/api/delete -d '{"model":"llama3.2:latest"}'
curl -s http://localhost:11434/api/embed -d '{"model":"nomic-embed-text","input":"hello"}'

# preload / unload
curl -s http://localhost:11434/api/generate -d '{"model":"llama3.2","keep_alive":-1}'
curl -s http://localhost:11434/api/generate -d '{"model":"llama3.2","keep_alive":0}'
```

## Response fields worth knowing

| Field | Meaning |
|---|---|
| `done` / `done_reason` | finished? (`stop`, `length`, …) |
| `prompt_eval_count` / `prompt_eval_duration` | prefill tokens / ns |
| `eval_count` / `eval_duration` | decode tokens / ns |
| `load_duration` | model load time (ns) |
| `message.thinking` | reasoning trace (when `think` is on) |
| `message.tool_calls` | requested function calls |
| `size_vram` (in `/api/ps`) | bytes in VRAM |

## Verified on

- Ollama API docs + OpenAPI spec, `0.40.14`, 2026-11-03.
