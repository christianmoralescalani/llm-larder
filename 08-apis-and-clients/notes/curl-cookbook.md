# curl cookbook — talking to a local LLM without an SDK

**Summary.** Every recipe below works against **any** of the three runtimes — set `BASE` once and go. `curl` is the fastest way to debug "is the server even alive?", test a new flag, or script a one-off. The patterns: health check, list models, chat, streaming (SSE), JSON mode, embeddings, multi-turn, tools, and images. One-page version: `cheatsheets/curl-cookbook.md`.

## Setup

```bash
# pick your runtime:
BASE=http://localhost:11434/v1   # Ollama
BASE=http://localhost:8080/v1    # llama.cpp
BASE=http://localhost:8000/v1    # vLLM
MODEL=qwen3.8                    # Ollama tag / --alias / served id
# if the server uses --api-key / OLLAMA_API_KEY:
AUTH=(-H "Authorization: Bearer sk-xxx")
```

## Health & models

```bash
curl -s $BASE/models -H "Content-Type: application/json"
# llama.cpp extra: /health  →  {"status":"ok"}
# Ollama native:    /api/version, /api/tags, /api/ps
```

## Chat (non-streaming)

```bash
curl -s $BASE/chat/completions "${AUTH[@]}" -H "Content-Type: application/json" -d '{
  "model": "'$MODEL'",
  "messages": [
    {"role": "system", "content": "You are terse."},
    {"role": "user", "content": "Explain KV cache in one line."}
  ],
  "stream": false,
  "temperature": 0.6,
  "max_tokens": 256
}' | python3 -m json.tool
```

## Streaming (SSE)

```bash
# -N disables buffering; watch the data: lines
curl -sN $BASE/chat/completions "${AUTH[@]}" -H "Content-Type: application/json" -d '{
  "model": "'$MODEL'",
  "messages": [{"role": "user", "content": "Count to 10, one word per line."}],
  "stream": true
}'
```

Output shape (all three runtimes):

```text
data: {"choices":[{"delta":{"role":"assistant","content":""}}]}
data: {"choices":[{"delta":{"content":"one"}}]}
data: {"choices":[{"delta":{"content":" two"}}]}
...
data: {"choices":[{"delta":{},"finish_reason":"stop"}]}
data: [DONE]
```

Parse it in a script:

```bash
curl -sN $BASE/chat/completions -d '{
  "model": "'$MODEL'", "stream": true,
  "messages": [{"role":"user","content":"Say hi"}]
}' | grep '^data: ' | sed 's/^data: //' | grep -v '\[DONE\]' \
  | python3 -c 'import json,sys
for line in sys.stdin:
    d = json.loads(line)
    t = d["choices"][0].get("delta", {}).get("content")
    if t: print(t, end="", flush=True)
print()'
```

## JSON mode

```bash
curl -s $BASE/chat/completions -H "Content-Type: application/json" -d '{
  "model": "'$MODEL'",
  "messages": [{"role":"user","content":"Population of France"}],
  "response_format": {"type": "json_object"},
  "stream": false
}'
# llama.cpp / vLLM also accept:
# "response_format": {"type":"json_schema","json_schema":{"name":"ans","schema":{...}}}
```

(For guaranteed-valid JSON in pipelines, prefer schema constraints — see [06 prompting](../06-sampling-and-prompting/notes/prompting-basics.md).)

## Embeddings

```bash
curl -s $BASE/embeddings -H "Content-Type: application/json" -d '{
  "model": "nomic-embed-text",
  "input": "what is a KV cache"
}'
```

## Multi-turn

The API is stateless — **you send the whole history every time**:

```bash
curl -s $BASE/chat/completions -H "Content-Type: application/json" -d '{
  "model": "'$MODEL'",
  "messages": [
    {"role":"user","content":"My name is Ana."},
    {"role":"assistant","content":"Nice to meet you, Ana!"},
    {"role":"user","content":"What is my name?"}
  ]
}'
```

## Tool calling

```bash
curl -s $BASE/chat/completions -H "Content-Type: application/json" -d '{
  "model": "'$MODEL'",
  "messages": [{"role":"user","content":"What is the weather in Lisbon?"}],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "Get current weather for a city",
      "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
      }
    }
  }],
  "tool_choice": "auto"
}'
# → choices[0].message.tool_calls[0].function.{name, arguments}
# then append the tool result as {"role":"tool","tool_call_id":"…","content":"…"} and call again
```

## Images

```bash
IMG=$(base64 -w0 photo.jpg)
curl -s $BASE/chat/completions -H "Content-Type: application/json" -d "{
  \"model\": \"'$MODEL'\",
  \"messages\": [{\"role\":\"user\",\"content\":[
    {\"type\":\"text\",\"text\":\"What is in this image?\"},
    {\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/jpeg;base64,$IMG\"}}
  ]}]
}"
```

## Debugging order

1. `curl $BASE/models` — is the server up, what's the exact model name?
2. Non-streaming one-liner — is the model answering at all?
3. Check `finish_reason` + `usage` — did it stop cleanly? how many tokens?
4. Stream — is SSE parsing working in your client?
5. Only then suspect the model/flags.

## Sources

- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [This repo — 04-llama-cpp/notes/api.md](../../04-llama-cpp/notes/api.md)
- [This repo — 03-ollama/notes/rest-api.md](../../03-ollama/notes/rest-api.md)

## My experiments

- (fill in: e.g. "the SSE one-liner above against :8080 (Qwen3.8) + Ollama + vLLM: byte-identical framing, only the `id` prefix differs; `response_format json_object` worked on all three; tool calls needed `tool_choice: auto` on Ollama or it answered in plain text")
