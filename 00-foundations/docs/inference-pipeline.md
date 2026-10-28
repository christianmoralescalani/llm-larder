# References — inference pipeline (prefill/decode)

- [HF — Inference optimization overview (compute vs memory bound)](https://huggingface.co/docs/transformers/perf_infer_gpu_one)
- [llama.cpp README — build, tools, performance notes](https://github.com/ggml-org/llama.cpp)
- [llama-bench — measure `pp` (prefill) vs `tg` (decode)](https://github.com/ggml-org/llama.cpp/tree/master/tools/llama-bench)
- [vLLM — PagedAttention & continuous batching](https://arxiv.org/abs/2309.06180)
- [FlashAttention — fusing attention, prefill speedup](https://arxiv.org/abs/2205.14135)
- [Speculative decoding — verifying several tokens per pass](https://arxiv.org/abs/2211.17192)
