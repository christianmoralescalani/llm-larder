# Offline inference — Python `LLM` class

**Summary.** No server needed: the `LLM` class runs batched inference in-process — pass a list of prompts, get back a list of outputs, with the same PagedAttention/continuous-batching engine as the server. `SamplingParams` controls decoding. In Docker, run Python inside the `vllm/vllm-openai` image with `--entrypoint python`. Use offline mode for: batch jobs (embeddings, evals, data generation), notebooks, and anything where an HTTP server is overhead.

## Minimal example

```python
from vllm import LLM, SamplingParams

prompts = [
    "Hello, my name is",
    "The capital of France is",
    "The future of AI is",
]
sampling_params = SamplingParams(temperature=0.8, top_p=0.95)

llm = LLM(model="facebook/opt-125m")
outputs = llm.generate(prompts, sampling_params)

for out in outputs:
    print(f"Prompt: {out.prompt!r}\nGenerated: {out.outputs[0].text!r}")
```

- `LLM(model=...)` downloads from Hugging Face (local paths work too, incl. GGUF); set `VLLM_USE_MODELSCOPE=True` for ModelScope.
- `llm.generate()` queues all prompts and runs them through the engine — **this is the batched path**; throughput, not latency, is the point.
- Each result is a `RequestOutput`: `.prompt`, `.outputs[i].text`, `.outputs[i].token_ids`, `.finished_reason`, `.metrics` (arrival/first-token/finish timestamps).

## Chat models

`generate()` does **not** apply the chat template — for Instruct/Chat models use `llm.chat()` with OpenAI-style messages:

```python
messages_list = [
    [{"role": "user", "content": "Who won the world series in 2020?"}],
    [{"role": "system", "content": "You are terse."},
     {"role": "user", "content": "Explain vLLM in one sentence."}],
]
outputs = llm.chat(messages_list, sampling_params)
```

Or apply the template yourself with the tokenizer and call `generate()`.

## SamplingParams (the common ones)

| Param | Default | Notes |
|---|---|---|
| `temperature` | 1.0 | 0 = greedy (deterministic) |
| `top_p` | 1.0 | nucleus sampling |
| `top_k` | -1 | -1 = disabled |
| `max_tokens` | None (until EOS) | cap output length |
| `min_p` | 0.0 | filter tokens below `min_p × P(max)` |
| `n` | 1 | independent samples per prompt |
| `stop` | — | str or list of strings/ids |
| `seed` | None | reproducible sampling |
| `logprobs` | None | per-token logprobs (0 = top-1 only) |
| `presence_penalty` / `frequency_penalty` | 0.0 | OpenAI-style penalties |
| `repetition_penalty` | 1.0 | |
| `ignore_eos` | False | keep generating past EOS |
| `detokenize` | True | `False` = return token ids only (faster for pipelines) |

`llm.generate()` also accepts `use_tqdm` (progress bar) and `sampling_params` as a **list** (per-prompt params).

## Run it in Docker

The image's entrypoint is `vllm serve`; override it for Python:

```bash
# 1) write the script
cat > /tmp/gen.py <<'EOF'
from vllm import LLM, SamplingParams

llm = LLM(model="Qwen/Qwen2.5-1.5B-Instruct", gpu_memory_utilization=0.9)
params = SamplingParams(temperature=0.7, max_tokens=128)
outs = llm.chat([[{"role": "user", "content": "Explain continuous batching in one sentence."}]], params)
print(outs[0].outputs[0].text)
EOF

# 2) run it (GPU + HF cache; script mounted read-only)
docker run --rm --gpus all --entrypoint python \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -v /tmp/gen.py:/gen.py:ro \
  vllm/vllm-openai:latest /gen.py
```

`LLM(...)` accepts the same engine args as `vllm serve` as kwargs: `tensor_parallel_size=2`, `max_model_len=32768`, `enable_prefix_caching=True`, `quantization="awq"`, …

## Pooling / embeddings / rerank

```python
llm = LLM(model="sentence-transformers/all-MiniLM-L6-v2", task="embed")
embeddings = llm.embed(["vLLM is fast", "Docker is portable"])

llm = LLM(model="BAAI/bge-reranker-v2-m3", task="score")
scores = llm.score(
    queries=["What is vLLM?"],
    documents=["A high-throughput LLM serving engine.", "A recipe for bread."],
)
```

`task` values: `embed`, `classify`, `score`, `reward` (or `--task` / `task=` kwarg). Same engine, different head.

## Gotchas

1. **`generate()` ≠ chat template** — use `llm.chat()` for Instruct models or your outputs will look like raw completion continuations.
2. **One model per `LLM`** — creating a second `LLM` loads a second copy (VRAM ×2); reuse one instance for many batches.
3. **`max_tokens=None`** — without a cap, a model that never emits EOS will run to `max_model_len`; always cap batch jobs.
4. **Engine args are kwargs, not flags** — `LLM(model=..., tensor_parallel_size=2)`, not `--tensor-parallel-size`.
5. **`--entrypoint python`** — the image runs as root; the script must be readable at the mounted path.
6. **Offline ≠ slower** — same engine as the server; the win is zero HTTP overhead and in-process control (logprobs, token ids, per-request params).

## Sources

- [Offline inference (quickstart)](https://docs.vllm.ai/en/stable/getting_started/quickstart/#offline-batched-inference)
- [LLM API](https://docs.vllm.ai/en/stable/api/vllm/entrypoints/llm/)
- [SamplingParams API](https://docs.vllm.ai/en/stable/api/vllm/sampling_params/)
- [Pooling models](https://docs.vllm.ai/en/stable/models/pooling/)

## My experiments

- (fill in: e.g. "batch of 1000 prompts through 1.5B offline vs served over HTTP: X min vs Y min; `detokenize=False` pipeline for eval token ids worked first try")

