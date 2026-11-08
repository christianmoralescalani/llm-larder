# Videos — 04-llama-cpp

Curated video list for this topic. Format:

- [Title](url) — why it is good, key timestamps, my notes

## Videos

- [Run AI Models Locally with llama.cpp](https://www.youtube.com/watch?v=ZR9S9zXm4ZU) — the deepest practical walkthrough: picking a quant on the Hub, the three Docker image variants (full/server/CUDA), `-ngl` layer offloading, `-c` context sizing, then switching `llama-cli` → `llama-server` and connecting an agent to it. Pairs with `notes/llama-server.md` and `cheatsheets/llama-cpp-docker.md`.
- [HOW to RUN AI MODELS locally using llama cpp and docker](https://www.youtube.com/watch?v=Fv-FZePp4as) — Docker-first: choosing the right image tag per backend (CPU/CUDA/ROCm), persisting a models volume, exposing the port, and consuming the server from a client. The shortest path to the setup in `testcommands`.
- [How to Run Local LLMs with Llama.cpp: Complete Guide](https://www.youtube.com/watch?v=EPYsP-l6z2s) — the quantization angle: when to quantize yourself vs. downloading a ready GGUF, and how `llama-quantize` fits in. Pairs with `notes/quantization.md`.
- [Run Fine-Tuned LLMs on CPU with llama.cpp](https://www.youtube.com/watch?v=vvrGfqd1oxE) — GGUF + llama.cpp on CPU-only hardware, and an honest comparison of llama.cpp as a server vs. heavier serving frameworks (no continuous batching) — useful for the "should I use vLLM instead?" question in [05-vllm](../05-vllm/cheatsheets/vllm-vs-llamacpp-vs-ollama.md).

## My notes

- (fill in: which ones I rewatched, timestamps worth bookmarking)
