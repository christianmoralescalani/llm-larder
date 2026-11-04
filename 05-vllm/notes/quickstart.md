# Quickstart — `vllm serve` in one command

**Summary.** One command gives you an OpenAI-compatible HTTP server: `vllm serve <HF model>` → `http://localhost:8000`. In Docker the command is after the image tag. From there you can use `curl`, any OpenAI SDK (point `base_url` at it), or vLLM's own `vllm chat`/`vllm complete` CLIs. The server applies the model's `generation_config.json` by default and the tokenizer's chat template — that's why chat "just works" out of the box.

## Serve (Docker)

```bash
# one command: OpenAI-compatible server on :8000
docker run --rm --gpus all -p 8000:8000 --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model Qwen/Qwen2.5-1.5B-Instruct
```

Native equivalent (pip install): `vllm serve Qwen/Qwen2.5-1.5B-Instruct`.

Useful additions:

```bash
  --host 0.0.0.0 --port 8000        # bind (defaults already fine in Docker)
  --served-model-name mymodel        # name clients use in "model"
  --api-key sk-...                   # require Bearer key on /v1*
  --max-model-len 32k                # cap context (k/K suffixes)
  --enable-prefix-caching            # on by default in V1; explicit for clarity
  --generation-config vllm           # ignore the model's generation_config.json
```

Watch `docker logs` until `Uvicorn running on http://0.0.0.0:8000` / startup complete.

## Talk to it

```bash
# list models
curl http://localhost:8000/v1/models

# completions
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","prompt":"San Francisco is a","max_tokens":7,"temperature":0}'

# chat completions
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model":"Qwen/Qwen2.5-1.5B-Instruct",
    "messages":[
      {"role":"system","content":"You are a helpful assistant."},
      {"role":"user","content":"Who won the world series in 2020?"}
    ]}'

# streaming (SSE)
curl -N http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen2.5-1.5B-Instruct","stream":true,
       "messages":[{"role":"user","content":"count to 5"}]}'

# with an API key
curl http://localhost:8000/v1/models -H "Authorization: Bearer sk-..."
```

## OpenAI SDK (drop-in)

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY",            # or your --api-key value
)
resp = client.chat.completions.create(
    model="Qwen/Qwen2.5-1.5B-Instruct",   # must match --served-model-name if set
    messages=[{"role": "user", "content": "Tell me a joke."}],
)
print(resp.choices[0].message.content)
```

Anything that speaks the OpenAI API (LangChain, LlamaIndex, Open WebUI, …) works by changing `base_url` only.

## vLLM's own CLIs (inside the image)

```bash
# interactive chat against a running server (override the entrypoint)
docker run --rm -it --gpus all --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest chat --url http://localhost:8000/v1

# one-liners
docker run --rm --gpus all --entrypoint vllm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest chat --quick "hi" --url http://<host>:8000/v1
```

(`vllm chat --stats` prints TTFT/throughput after each response — handy for a first feel.)

## What's on by default (and how to change it)

| Behavior | Default | Override |
|---|---|---|
| Chat template | from the model's tokenizer | `--chat-template <file/jinja>` |
| Sampling params | model's `generation_config.json` | `--generation-config vllm` (vLLM defaults) or `--override-generation-config '{"temperature":0.5}'` |
| Context length | model max (or what fits in VRAM) | `--max-model-len 32k` |
| Prefix caching | **on** (V1) | `--disable-prefix-caching` |
| Port / host | `8000` / `0.0.0.0` | `--port`, `--host`, `--uds /tmp/vllm.sock` |

## Gotchas

1. **"model" must match** — the `model` field in requests must equal `--served-model-name` (or the `--model` value if you didn't set one); mismatch = 404/`model_not_found`.
2. **First request is slow** — weights load and CUDA graphs capture at startup; the first chat after boot is the warmup.
3. **`generation_config.json` surprises** — some repos pin `max_new_tokens`/temperature; `--generation-config vllm` if defaults look "wrong".
4. **`/v1/completions` has no `suffix`** and `/v1/chat/completions` ignores `user` — the OpenAI-compat surface is broad but not 100%.
5. **`--api-key` is partial auth** — it guards `/v1`, `/v2`, `/inference` only; `/invocations` and `/metrics` stay open.
6. **One model per server** — for a second model, run a second container on another port.

## Sources

- [Quickstart](https://docs.vllm.ai/en/stable/getting_started/quickstart/)
- [OpenAI-Compatible Server](https://docs.vllm.ai/en/stable/serving/online_serving/openai_compatible_server/)
- [CLI guide: chat / complete](https://docs.vllm.ai/en/stable/cli/)

## My experiments

- (fill in: e.g. "served Qwen3.8-27B (HF FP8) in Docker: `vllm chat --stats` showed TTFT ~X ms / Y tok/s on 1.5B; pointed Open WebUI at :8000 — worked with zero config changes")
