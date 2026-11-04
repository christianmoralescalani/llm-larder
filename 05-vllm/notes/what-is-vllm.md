# What vLLM is

**Summary.** vLLM is a Python library + OpenAI-compatible server for high-throughput LLM inference, originally from UC Berkeley's Sky Computing Lab (PagedAttention paper, SOSP 2023). It's the "production" end of this repo's toolchain: where llama.cpp/Ollama optimize for one user on one machine, vLLM optimizes for *many concurrent requests* on one or more GPUs. The two ideas behind its speed: **PagedAttention** (KV cache managed in blocks, like OS virtual memory) and **continuous batching** (new requests join/leave the GPU batch every iteration, not at request boundaries).

## The problem it solves

Naive serving runs one request at a time (or a static batch that waits for the slowest member to finish). Two wastes:

1. **KV cache fragmentation** — each sequence's KV cache must be a contiguous allocation; reserved-but-unused space ("internal fragmentation") can waste 60–80% of GPU memory.
2. **Idle GPU between requests** — a request that finishes early leaves its slot empty until the whole batch finishes.

## PagedAttention

The KV cache is split into fixed-size **blocks** (default 16 tokens) and allocated on demand, exactly like virtual memory pages:

- Logical blocks (per sequence) map to physical blocks in a shared pool.
- No pre-reservation: a sequence grows block-by-block, so unused memory ≈ 0.
- Blocks of identical prefixes can be **shared** between sequences (copy-on-write) — this is what makes prefix caching almost free.
- Preemption (evicting a running request's blocks to make room) works because blocks are addressable units; the evicted request is recomputed on reschedule.

## Continuous batching

The scheduler rebuilds the GPU batch **every iteration** (every token step):

- A request whose generation finished drops out immediately; a waiting request that now fits steps in — no batch boundaries.
- **Chunked prefill**: long prompts are sliced into chunks and interleaved with decode steps, so one 32k prompt doesn't stall everyone's token streaming.
- Net effect: the GPU stays saturated, and per-request latency stays close to the single-request case even under load. (Anyscale's writeup measured ~23× throughput over static batching with lower p50 latency.)

## What else makes it fast

- **CUDA/HIP graphs** (piecewise and full) to kill kernel-launch overhead; `torch.compile` for fusion.
- **Optimized kernels**: FlashAttention / FlashInfer / TRTLLM-GEN / FlashMLA attention; CUTLASS GEMM/MoE kernels.
- **Prefix caching** (hash-based, on by default in V1): shared system prompts / RAG contexts skip recomputation.
- **Speculative decoding** (n-gram, suffix, EAGLE, MTP, draft models): lower inter-token latency under low-to-mid QPS.
- **Quantization**: FP8, NVFP4/MXFP4, INT8/INT4, AWQ, GPTQ, GGUF, bitsandbytes, and more.
- **Distributed inference**: tensor / pipeline / data / expert / context parallelism.

## Position in this repo

| Tool | Best for |
|---|---|
| Ollama (03) | zero-friction local use, `ollama run` |
| llama.cpp (04) | single-user max control, GGUF quants, CPU/hybrid, MTP drafts |
| **vLLM (05)** | serving many concurrent requests, OpenAI-compatible API, multi-GPU, HF weights (incl. GGUF) |

All three can run the same model; the choice is workload shape (see `cheatsheets/vllm-vs-llamacpp-vs-ollama.md`).

## Gotchas

1. **"Fast" means throughput** — at 1 request, vLLM isn't necessarily faster than llama.cpp per token; its advantage appears as concurrency rises.
2. **V1 engine** — current vLLM is the V1 rewrite (single scheduler loop, no CPU swap). Old blog posts about `--swap-space`/swapped requests are obsolete.
3. **Weights come from Hugging Face** — `--model` takes an HF repo ID or a local path (safetensors *or* GGUF). Your llama.cpp GGUFs load directly; no conversion needed.
4. **Heavy startup** — first run compiles/captures CUDA graphs (minutes on Hopper+); `vllm download-kernels` (pip) and a mounted compile cache (Docker) fix most of it.
5. **One model per instance** — a `vllm serve` process serves one model; scale out with replicas, not by stacking models.

## Sources

- [vLLM docs home](https://docs.vllm.ai/en/stable/)
- [PagedAttention blog post](https://blog.vllm.ai/2023/06/20/vllm.html)
- [vLLM paper (SOSP 2023)](https://arxiv.org/abs/2309.06180)
- [Continuous batching (Anyscale)](https://www.anyscale.com/blog/continuous-batching-llm-inference)
- [Automatic Prefix Caching design](https://docs.vllm.ai/en/stable/design/prefix_caching/)

## My experiments

- (fill in: e.g. "served X with 20 concurrent chat requests: llama.cpp tg = Y t/s per stream, vLLM aggregate = Z t/s with p50 TTFT W s — first time I felt continuous batching")
