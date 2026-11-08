# References — 06-sampling-and-prompting

Official docs and the tools this section is based on.

## Sampling & generation

- [HF — Generation strategies (temperature, top_p, top_k, min_p, penalties)](https://huggingface.co/docs/transformers/en/generation_strategies)
- [HF — Generation parameters (API reference)](https://huggingface.co/docs/transformers/en/internal/generation_utils)
- [Nucleus sampling paper (Holtzman et al. 2019)](https://arxiv.org/abs/1904.09751) — origin of top_p + typical sampling
- [CTRL paper (Keskar et al. 2019)](https://arxiv.org/abs/1909.05858) — origin of repetition penalty
- [OpenAI — Prompt engineering guide](https://platform.openai.com/docs/guides/prompt-engineering) — few-shot, CoT, patterns
- [OpenAI — API reference (sampling fields)](https://platform.openai.com/docs/api-reference)

## llama.cpp (sampling, templates, reasoning)

- [Server README — full flag reference (sampling section, `--jinja`, `--reasoning*`)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [common/sampling.cpp — sampler chain implementation](https://github.com/ggml-org/llama.cpp/blob/master/common/sampling.cpp)
- [llama-bench README](https://github.com/ggml-org/llama.cpp/blob/master/tools/llama-bench/README.md)
- [SPEED-Bench client (server-level benchmarking)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/bench/speed-bench/README.md)

## Ollama

- [Modelfile reference (`PARAMETER`, `SYSTEM`, `TEMPLATE`)](https://docs.ollama.com/modelfile)
- [API reference (`options`, `think`, timing fields)](https://docs.ollama.com/api)
- [Thinking (blog: CLI + API controls)](https://ollama.com/blog/thinking)

## vLLM

- [SamplingParams API](https://docs.vllm.ai/en/stable/api/vllm/sampling_params/)
- [Reasoning outputs (`--reasoning-parser`, `include_reasoning`, template kwargs)](https://docs.vllm.ai/en/latest/features/reasoning_outputs)
- [Structured outputs (guided JSON / grammar)](https://docs.vllm.ai/en/stable/features/structured_outputs/)
- [Benchmark CLI](https://docs.vllm.ai/en/stable/benchmarking/cli/)
- [Metrics design doc](https://docs.vllm.ai/en/stable/design/metrics/)

## Model-family sampling guides

- [Qwen — sampling parameters & vLLM deployment (thinking modes)](https://qwen.readthedocs.io/en/latest/usage/sampling_params.html)
- [Llama 3 — recommended generation settings](https://huggingface.co/docs/transformers/en/model_doc/llama3)
- [Mistral — generation parameters](https://huggingface.co/mistralai/Mistral-7B-v0.3)

## Chat templates

- [HF — Chat templating (canonical guide)](https://huggingface.co/docs/transformers/main/en/chat_templating)
- [HF — Conversations basics (message format)](https://huggingface.co/docs/transformers/main/en/conversations)
- [ChatML format (GPT-3.5/4 style)](https://arxiv.org/abs/2211.09110)

## In-repo

- `00-foundations/notes/system-prompts-and-chat-templates.md` — the concept (this folder is the practical side)
- `00-foundations/notes/inference-pipeline.md` — prefill vs decode (the two phases behind TTFT/decode speed)
- `03-ollama/notes/rest-api.md` — Ollama API (where `options` and `think` live)
- `04-llama-cpp/notes/llama-server.md` — full llama-server flags (sampling, jinja, reasoning)
- `04-llama-cpp/notes/benchmarking.md` — `llama-bench` deep dive
- `05-vllm/notes/benchmarking.md` — `vllm bench` deep dive
- `05-vllm/notes/offline-inference.md` — `SamplingParams` table
