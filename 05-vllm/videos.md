# Videos — 05-vllm

Curated video list for this topic. Format:

- [Title](url) — why it is good, key timestamps, my notes

## Videos

- [vLLM Serving Tutorial: High-Performance LLM Inference with Paged Attention and LoRA — Ready Tensor](https://www.youtube.com/watch?v=WBqjB4UMXXE) — the concept + practice pass: what vLLM is, PagedAttention, continuous batching, then install → `vllm serve` → LoRA routing → throughput comparison. Key timestamps: 0:00 what is vLLM, 0:38 key features, 2:30 installing, 3:25 launching the server, 4:00 serving LoRA adapters, 7:25 throughput gains.
- [How-to Install vLLM and Serve AI Models Locally — Fahad Mirza](https://www.youtube.com/watch?v=9gX5bgtvuUU) — the plain-English install: venv → `pip install vllm` → `vllm serve` → hitting the OpenAI-compatible endpoint from Python, with live VRAM usage on screen. Pairs with `notes/quickstart.md`.
- [vLLM: Easily Deploying & Serving LLMs](https://www.youtube.com/watch?v=q5IF2PHA5SA) — covers both modes: the offline `LLM` class (`SamplingParams`, `llm.generate`) and `vllm serve` with `--gpu-memory-utilization` and `--api-key`. The offline section pairs with `notes/offline-inference.md`.
- [vLLM: Introduction and easy deploying](https://www.youtube.com/watch?v=p1n4tgQta2U) — short intro to the engine's history (UC Berkeley Sky Lab) and an easy first deployment. Good 10-minute primer before the two longer videos above.

## My notes

- (fill in: which ones I rewatched, timestamps worth bookmarking)
