# Videos — 02-hardware-and-vram

Curated video list for this topic. Format:

- [Title](url) — why it is good, key timestamps, my notes

## Videos

- [Find the amount of VRAM required to run a Large Language Model locally — Apex Machine](https://www.youtube.com/watch?v=soM6Jw3wUNI) — walks through the "LLM VRAM & Performance Calculator": weights + KV + overhead, inference vs fine-tuning memory, quantization (FP16 vs Q4) impact. Pairs with `notes/vram-formula.md`.
- [Formula to Calculate GPU Memory for Serving LLMs Locally](https://www.youtube.com/watch?v=LEkdoV5nNW4) — short, focused on the sizing formula for local serving: how many GPUs / how much VRAM for a given model. Good 10-minute companion to `cheatsheets/vram-budget.md`.
- [Local AI just leveled up… Llama.cpp vs Ollama — Alex Ziskind](https://www.youtube.com/watch?v=2t9XrPcAiHg) — practical hardware/runtime comparison: GGUF setup, concurrency limits, parallel chats (10:44 Ollama concurrency, 11:48 llama.cpp parallel). Useful context for the offload/batch tradeoffs in `notes/kv-cache-math.md`.
- [Learn Ollama in 15 Minutes — Tech With Tim](https://www.youtube.com/watch?v=UtSSMs6ObqY) — install → run → API in 15 min (0:37 install, 1:33 running models, 6:43 HTTP API). The fastest path to *seeing* VRAM usage with `ollama ps` on your own card.
- [Run LLMs Locally in Minutes with Ollama — DeepSeek, LLaMA & More!](https://www.youtube.com/watch?v=_bXH9OyshZY) — 4-minute setup incl. DeepSeek; good for testing which quant your VRAM tier actually runs.
- [How To EASILY Run Open Source AI Models Locally (Tutorial)](https://www.youtube.com/watch?v=8a6NctiECJA) — Ollama + Pinokio walkthrough; chapters from 2:45 cover extra tools. Handy if you want a GUI before diving into flags.

## To curate

- _(search for: "RTX 3090 vs 4090 LLM benchmark", "Apple Silicon LLM tokens per second M4 Max", "GPU memory bandwidth explained", "multi-GPU LLM inference tensor parallelism", "llama.cpp n-gpu-layers benchmark")_

## My notes

- (fill in: which ones I rewatched, timestamps worth bookmarking)
