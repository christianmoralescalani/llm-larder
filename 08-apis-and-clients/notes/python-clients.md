# Python clients — the `openai` SDK pointed at local

**Summary.** The official `openai` Python SDK works against all three local runtimes unchanged — set `base_url` and a dummy `api_key`, done. This note covers the sync client, streaming, the async client, a multi-runtime config pattern, and the small glue functions (chat, structured output, RAG query) you'll actually reuse. Keep it boring: one client object, one model name, config-driven.

## Install & setup

```bash
pip install openai
```

```python
from openai import OpenAI

# Ollama
client = OpenAI(base_url="http://localhost:11434/v1", api_key="none")
MODEL = "qwen3:8b"

# llama.cpp
# client = OpenAI(base_url="http://localhost:8080/v1", api_key="none")
# MODEL = "qwen3.8"

# vLLM
# client = OpenAI(base_url="http://localhost:8000/v1", api_key="none")
# MODEL = "Qwen/Qwen3-8B"
```

## The three core calls

```python
# 1. chat
r = client.chat.completions.create(
    model=MODEL,
    messages=[{"role": "user", "content": "Explain KV cache in one line."}],
    temperature=0.6,
    max_tokens=256,
)
print(r.choices[0].message.content)
print(r.usage)  # prompt_tokens, completion_tokens, total_tokens

# 2. streaming
stream = client.chat.completions.create(
    model=MODEL,
    messages=[{"role": "user", "content": "Count to 10."}],
    stream=True,
)
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
print()

# 3. embeddings
e = client.embeddings.create(model="nomic-embed-text", input="what is a KV cache")
vec = e.data[0].embedding  # list of floats
```

## Async

```python
from openai import AsyncOpenAI
import asyncio

client = AsyncOpenAI(base_url="http://localhost:8080/v1", api_key="none")

async def ask(prompt: str) -> str:
    r = await client.chat.completions.create(
        model="qwen3.8",
        messages=[{"role": "user", "content": prompt}],
    )
    return r.choices[0].message.content

async def main():
    # fan out independent questions concurrently
    results = await asyncio.gather(*[ask(p) for p in QUESTIONS])

asyncio.run(main())
```

Async matters when you fan out (batch jobs, RAG over many chunks); for a single chat loop the sync client is fine.

## Multi-runtime config pattern

```python
import os
from openai import OpenAI

RUNTIMES = {
    "ollama":   ("http://localhost:11434/v1", "qwen3:8b"),
    "llamacpp": ("http://localhost:8080/v1",  "qwen3.8"),
    "vllm":     ("http://localhost:8000/v1",  "Qwen/Qwen3-8B"),
}

def get_client(runtime: str = os.environ.get("LLM_RUNTIME", "llamacpp")):
    base_url, model = RUNTIMES[runtime]
    return OpenAI(base_url=base_url, api_key=os.environ.get("LLM_API_KEY", "none")), model

client, MODEL = get_client()
```

One script, three backends, zero code changes — that's the whole point of the OpenAI-compatible layer.

## Reusable glue

```python
def chat(messages, **kw) -> str:
    """One-shot chat with sane defaults + finish_reason check."""
    r = client.chat.completions.create(
        model=MODEL, messages=messages,
        temperature=kw.pop("temperature", 0.6),
        max_tokens=kw.pop("max_tokens", 1024),
        **kw,
    )
    if r.choices[0].finish_reason == "length":
        raise RuntimeError("output truncated — raise max_tokens")
    return r.choices[0].message.content

def chat_json(messages, schema: dict, **kw) -> dict:
    """Structured output with validation + one retry."""
    import json
    for attempt in range(2):
        text = chat(messages, **kw)
        try:
            return json.loads(text)
        except json.JSONDecodeError:
            messages = messages + [
                {"role": "assistant", "content": text},
                {"role": "user", "content": "That was not valid JSON. Reply with only the JSON object."},
            ]
    raise ValueError("model failed to produce valid JSON twice")

def rag_answer(chunks: list[str], question: str) -> str:
    """Minimal RAG: stuff retrieved chunks, ask, cite nothing fancy."""
    context = "\n\n".join(f"[{i+1}] {c}" for i, c in enumerate(chunks))
    return chat([
        {"role": "system", "content":
         "Answer using only the context. If it's not there, say so.\n\n"
         f'Context:\n"""\n{context}\n"""\n'},
        {"role": "user", "content": question},
    ])
```

Notes:
- For **guaranteed** JSON, use `response_format={"type":"json_schema", ...}` (llama.cpp/vLLM) instead of the retry loop — `chat_json` above is the portable fallback.
- **Reasoning models**: check `getattr(msg, "reasoning_content", None)` (llama.cpp/vLLM) or Ollama's `thinking` field; `content` may be empty until thinking ends.
- **Retries**: wrap client creation in a retry for cold starts (server still loading the model → 503/connection refused); a simple `for i in range(5): sleep(2 ** i)` before first call is usually enough.

## Gotchas

1. **`api_key` is required by the SDK** even when the server ignores it — pass `"none"`.
2. **Timeouts** — the SDK default can be short for big prefills; set `timeout=120` (or higher for 27B+ models with long prompts).
3. **`max_tokens`** is the portable name; `max_completion_tokens` is the newer OpenAI name — stick with `max_tokens` locally.
4. **Streaming + async** — `async for chunk in stream` with `stream=True` on `AsyncOpenAI`; don't mix sync/async clients.
5. **Model name 404s** — the #1 "it's broken" is a wrong model name per runtime (see the config pattern above).

## Sources

- [OpenAI Python SDK docs](https://github.com/openai/openai-python)
- [Ollama — OpenAI compatibility](https://docs.ollama.com/api)
- [vLLM — OpenAI compatibility](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/)
- [This repo — 06 prompting basics (RAG patterns)](../06-sampling-and-prompting/notes/prompting-basics.md)

## My experiments

- (fill in: e.g. "one script against all three runtimes via the config pattern: identical results modulo sampling noise (fixed seed); `timeout=120` needed for Qwen3.8-27B with 8k-prompt RAG on :8080; `chat_json` retry loop never fired once llama.cpp's `json_schema` was available — kept it only for Ollama")
