# Cheatsheet — curl cookbook (any local runtime)

**TL;DR.** Set `BASE` + `MODEL` once; every recipe below works against Ollama, llama.cpp, and vLLM. Deep dive: `notes/curl-cookbook.md`.

```bash
BASE=http://localhost:11434/v1   # Ollama | :8080/v1 llama.cpp | :8000/v1 vLLM
MODEL=qwen3.8                    # tag / --alias / served id
```

## Health & models

```bash
curl -s $BASE/models
curl -s http://localhost:8080/health       # llama.cpp: {"status":"ok"}
curl -s http://localhost:11434/api/version # Ollama native
```

## Chat

```bash
curl -s $BASE/chat/completions -H "Content-Type: application/json" -d '{
  "model": "'$MODEL'",
  "messages": [
    {"role": "system", "content": "You are terse."},
    {"role": "user", "content": "Explain KV cache in one line."}
  ],
  "stream": false, "temperature": 0.6, "max_tokens": 256
}' | python3 -m json.tool
```

## Streaming (SSE) — extract just the text

```bash
curl -sN $BASE/chat/completions -d '{
  "model": "'$MODEL'", "stream": true,
  "messages": [{"role":"user","content":"Count to 10."}]
}' | grep '^data: ' | sed 's/^data: //' | grep -v '\[DONE\]' \
  | python3 -c 'import json,sys
for l in sys.stdin:
    t = json.loads(l)["choices"][0].get("delta",{}).get("content")
    if t: print(t, end="", flush=True)
print()'
```

## JSON mode

```bash
curl -s $BASE/chat/completions -d '{
  "model": "'$MODEL'",
  "messages": [{"role":"user","content":"Population of France"}],
  "response_format": {"type":"json_object"}
}'
# llama.cpp/vLLM: "response_format":{"type":"json_schema","json_schema":{...}}
```

## Embeddings

```bash
curl -s $BASE/embeddings -d '{"model":"nomic-embed-text","input":"what is a KV cache"}'
```

## Tool calling

```bash
curl -s $BASE/chat/completions -d '{
  "model": "'$MODEL'",
  "messages": [{"role":"user","content":"Weather in Lisbon?"}],
  "tools": [{"type":"function","function":{
    "name":"get_weather","description":"Current weather for a city",
    "parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}
  }}],
  "tool_choice":"auto"
}'
# → message.tool_calls[0].function.{name,arguments}; reply with role:"tool" + tool_call_id
```

## Images

```bash
IMG=$(base64 -w0 photo.jpg)
curl -s $BASE/chat/completions -d "{
  \"model\": \"'$MODEL'\",
  \"messages\": [{\"role\":\"user\",\"content\":[
    {\"type\":\"text\",\"text\":\"What is in this image?\"},
    {\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/jpeg;base64,$IMG\"}}
  ]}]
}"
```

## Debugging order

1. `curl $BASE/models` — server up? exact model name?
2. Non-streaming one-liner — does it answer at all?
3. Check `finish_reason` + `usage` — clean stop? token count?
4. Stream — is SSE parsing working?
5. Only then suspect the model/flags.

Verified against Ollama 0.40.x, llama.cpp master (v0.8.0), vLLM 0.9.x — 2026-11-08.
