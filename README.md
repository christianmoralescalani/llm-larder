# LocalLLM

Notes, videos, slides, cheatsheets, and docs about **running LLMs locally**.

## What's here

| Topic | Folder |
|---|---|
| Foundations (tokens, parameters, MoE, MTP, quantization) | [`00-foundations/`](00-foundations/) |
| Hugging Face (searching & downloading models) | [`01-huggingface/`](01-huggingface/) |
| Hardware & VRAM calculation | [`02-hardware-and-vram/`](02-hardware-and-vram/) |
| Ollama (install, commands, config) | [`03-ollama/`](03-ollama/) |
| llama.cpp (all tools & flags) | [`04-llama-cpp/`](04-llama-cpp/) |
| vLLM (serving) | [`05-vllm/`](05-vllm/) |
| Sampling & prompting | [`06-sampling-and-prompting/`](06-sampling-and-prompting/) |
| Multimodal / vision models | [`07-multimodal-vision/`](07-multimodal-vision/) |
| APIs & clients (OpenAI-compatible, frontends) | [`08-apis-and-clients/`](08-apis-and-clients/) |
| Glossary & curated links | [`09-glossary-and-links/`](09-glossary-and-links/) |

## How to navigate

- **Start here if you're new**: [Structure.md](Structure.md) → follow the roadmap in section 4.
- **Need a command fast?** → look in the topic's `cheatsheets/` folder.
- **Learning a topic deeply?** → `notes/` + `videos.md` + `slides/`.
- **Official references** → `docs/` in each topic.

## Suggested learning path

1. `00-foundations` + `02-hardware-and-vram` (concepts + VRAM math)
2. `01-huggingface` (find & download models)
3. `03-ollama` (fastest way to run something)
4. `04-llama-cpp` (max control, GGUF workflow)
5. `05-vllm` (high-performance serving)

## Status

See [Structure.md](Structure.md) — every topic has a checklist with `[ ] / [~] / [x]` markers.
