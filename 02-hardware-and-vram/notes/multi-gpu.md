# Multi-GPU: splitting models across cards

**Summary.** When a model doesn't fit on one card, split it. There are two ways: **layer split** (pipeline — GPU 0 runs layers 0–39, GPU 1 runs 40–79; only small activations cross PCIe; llama.cpp's default) and **tensor parallelism** (each layer is cut across GPUs; large intermediate tensors are exchanged per layer; vLLM's default; wants NVLink). The classic consumer setup: **70B Q4_K_M (42.7 GB) on 2×24 GB** — fits with room for 8K context, decodes at ~15–25 t/s.

## When to go multi-GPU

| Goal | Setup |
|---|---|
| 70B Q4_K_M (42.7 GB) | 2×24 GB (48 GB total) |
| 70B Q5_K_M (50.4 GB) | 2×24 GB (tight) or 2×32 GB |
| 70B Q8_0 (75 GB) | 3×32 GB or 4×24 GB |
| 106B MoE Q4 (~60 GB) | 2×32 GB |
| 235B MoE Q3_K_M (~110 GB) | 4×32 GB |
| 405B Q2 (~156 GB) | 4×32–48 GB / 8×24 GB |

Also useful without a capacity need: more cards = more prefill compute (faster TTFT on long prompts).

## The two splitting strategies

### 1. Layer split (pipeline parallelism)

Layers 0..N/2 on GPU 0, N/2..N on GPU 1. Per token, the *activation* (a hidden-size vector, ~tens of KB) crosses PCIe once per boundary. Negligible cost — **PCIe is fine**.

- llama.cpp: default behavior when `--n-gpu-layers` exceeds one GPU's capacity.
- vLLM: `--pipeline-parallel-size`.

### 2. Tensor parallelism

Every layer is split; all GPUs compute every token and exchange large intermediate tensors (hidden × batch) **per layer**. This is bandwidth-hungry:

- vLLM: `--tensor-parallel-size` (default strategy).
- Wants **NVLink** (consumer: only RTX 3090 has it). Over PCIe it still works, with a speed penalty that grows with batch size.

Rule of thumb: **single user → layer split is fine; high-throughput serving → tensor parallel + NVLink if you have it.**

## llama.cpp (layer split)

```bash
# 70B Q4_K_M on 2×24 GB: all 80 layers on GPU, split evenly
llama-server -m Llama-3.1-70B-Q4_K_M.gguf \
  --n-gpu-layers 80 \
  --main-gpu 0 \
  --tensor-split 1,1 \
  --ctx-size 8192
```

- `--n-gpu-layers 80`: put every layer on GPU (across both cards). If it doesn't fit, lower it — the remainder goes to CPU (check the log: `offloaded 80/80 layers to GPU`).
- `--main-gpu 0`: embeddings + output head on GPU 0.
- `--tensor-split 2,1`: give GPU 0 double the layers (e.g. 32 GB + 16 GB cards — split proportionally to VRAM).
- OOM on load → reduce `--n-gpu-layers` or `--ctx-size`.

## vLLM (tensor parallel by default)

```bash
# 70B on 2×24 GB: needs a quant — fp16 (141 GB) doesn't fit even split
vllm serve meta-llama/Llama-3.1-70B \
  --tensor-parallel-size 2 \
  --quantization fp8 \
  --max-model-len 8192
```

- TP divides weights across GPUs: 70B fp8 ≈ 70 GB total → ~35 GB per card ✓ on 2×24. 70B fp16 → ~70 GB per card ✗.
- With AWQ/GPTQ int4 repos: ~40 GB total → 20 GB per card, comfortable.
- `--pipeline-parallel-size N` for layer split instead (rarely needed on consumer hardware).

## Ollama

```bash
export OLLAMA_NUM_GPU=2          # allow Ollama to use both cards
export CUDA_VISIBLE_DEVICES=0,1  # which cards
ollama run llama3.1:70b
```

Ollama does layer split automatically (it's llama.cpp underneath); `OLLAMA_NUM_GPU` controls how many cards it may use.

## Worked example: 70B Q4_K_M on 2×24 GB

```text
weights (Q4_K_M)        42.7 GB   → ~21.4 GB per card
KV cache @ 8K (fp16)    2.5 GB
overhead                ~1.0 GB
─────────────────────────────────
total                  ~46.2 GB  ≤ 48 GB ✓ (tight but fits)
```

Expect ~15–25 t/s decode (bandwidth-limited; the PCIe activation hops are cheap). At 32K context the KV grows to 10.2 GB → 53.9 GB total → **doesn't fit**; drop to q8_0 cache (5.1 GB) or 16K ctx.

## Pitfalls

- **Uneven split**: `--tensor-split 1,1` on a 32 GB + 16 GB box OOMs the small card. Split proportionally (`2,1`).
- **Silent CPU fallback**: if layers don't fit, the rest go to CPU and speed collapses. Always read the `offloaded X/Y layers` log line.
- **PCIe generation**: x16 Gen4 ≈ 32 GB/s per direction. Fine for layer split; a bottleneck for tensor parallel at high batch.
- **MoE across GPUs**: experts distribute across cards; a single token's active experts may live on different GPUs — llama.cpp handles it, but expect slightly lower efficiency than dense.
- **One card idling** in benchmarks usually means the split is wrong or one card is doing prefill while the other waits.

## Sources

- llama.cpp docs: `--n-gpu-layers`, `--main-gpu`, `--tensor-split` (see [docs/references.md](../docs/references.md))
- vLLM docs: tensor/pipeline parallelism, `--quantization`
- Ollama docs: `OLLAMA_NUM_GPU`, `CUDA_VISIBLE_DEVICES`
- [vram-formula.md](vram-formula.md) for the 70B budget math

## My experiments

- (fill in: e.g. "70B Q4_K_M on 2×3090: 80/80 layers offloaded, 19 t/s decode, 2200 t/s prefill at 8K ctx")
