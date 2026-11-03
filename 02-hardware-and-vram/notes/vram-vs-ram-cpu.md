# VRAM vs RAM vs CPU: what happens when the model doesn't fit

**Summary.** There are three tiers of running a model: **everything in VRAM** (full speed), **partial offload** (some layers on CPU — speed drops to the slowest segment), and **CPU-only** (speed limited by system RAM bandwidth, not compute). The key number for the CPU tiers is your **RAM bandwidth** (e.g. dual-channel DDR5-5600 ≈ 90 GB/s): `tok/s ≈ bandwidth × 0.6–0.8 ÷ model_bytes`. One exception: **MoE models run fast on CPU** because only a few GB of active weights are read per token.

## The three tiers

| Tier | Where weights live | Speed (8B Q4, 4.9 GB) | Speed (70B Q4, 42.7 GB) |
|---|---|---|---|
| All in VRAM | GPU | ~50–150 t/s (bandwidth-dependent) | ~15–25 t/s on 2×24 GB |
| Partial offload | split GPU/CPU | ~10–30 t/s | ~5–10 t/s |
| CPU only | RAM (DDR5 dual) | ~8–15 t/s | ~2–4 t/s |

*(Rough ranges from community benchmarks; your runtime and model mix change them. Measure with `llama-bench`.)*

## Partial offload: the speed cliff

`--n-gpu-layers N` puts the first N layers on the GPU, the rest on the CPU. Every generated token must pass through **both** segments, so throughput is dominated by the slowest one.

Example, 70B Q4_K_M (42.7 GB) with a 24 GB GPU:

| Setup | Est. decode |
|---|---|
| 2×24 GB, all 80 layers on GPU | ~15–25 t/s |
| 1×24 GB (≈50 layers) + CPU | ~8–12 t/s |
| CPU only (DDR5-5600 dual) | ~2–4 t/s |

Partial offload is *usable* (8–12 t/s is chat-speed), but it's a trap if you expected GPU speed. The runtime will happily load 50/80 layers on the GPU and the rest on CPU — check the log line `offloaded X/Y layers to GPU` and make sure Y is what you intended.

## CPU-only inference

Decode on CPU is **memory-bandwidth-bound**: each token re-reads the (active) weights from RAM once.

```text
tok/s ≈ RAM_bandwidth × efficiency (0.6–0.8) ÷ model_bytes
```

| RAM | Bandwidth (dual-channel) | 8B Q4 (4.9 GB) | 70B Q4 (42.7 GB) |
|---|---|---|---|
| DDR4-3200 ×2 | ~51 GB/s | ~5–8 t/s | ~0.6–1 t/s |
| DDR5-5600 ×2 | ~90 GB/s | ~8–15 t/s | ~2–4 t/s |
| DDR5-6400 ×2 (8-channel EPYC/Threadripper) | ~200–400 GB/s | ~20–40 t/s | ~10–25 t/s |

Rules:

- **Two DIMMs minimum** — single-channel halves your bandwidth and your tok/s.
- AVX2/AVX-512/AMX matter for prefill (prompt processing), less for decode.
- 70B dense on a normal desktop CPU is 2–4 t/s: it works, it's slow, it's fine for batch jobs.

## The MoE exception

MoE models read only their **active** experts per token (e.g. Qwen3-30B-A3B: 18.6 GB file, but only ~2 GB of active weights per token):

| Model | File (Q4) | CPU decode (DDR5 ~90 GB/s) |
|---|---|---|
| Qwen3-30B-A3B | 18.6 GB | ~30–60 t/s |
| gpt-oss-120B (MXFP4) | ~63 GB | ~15–30 t/s |

This is why a 30B-A3B MoE on a decent CPU beats a 13B dense model on the same CPU, and why MoE is the go-to choice for CPU/partial-offload builds.

## mmap, mlock, and swapping

- **mmap** (default): the OS pages weights in from the file on demand. You need **system RAM ≥ model file size** for good speed; if RAM is smaller, pages get evicted and re-read from disk per token → collapse to <1 t/s.
- **`--mlock`**: pins weights in RAM, prevents page-out. Use it when RAM ≥ file size and you want deterministic speed.
- **macOS**: unified memory means "RAM" is shared; if the model + OS exceed physical RAM, macOS pages out to the SSD (swap) — same collapse. Keep total usage under ~75% of physical RAM (see [apple-silicon.md](apple-silicon.md)).
- **`--no-mmap`**: loads everything up front; useful for benchmarking or when the file is on a slow disk.

## Decision rules

1. **Fit it all in VRAM** if you can — that's the only tier where prefill is also fast.
2. **Can't fit? Choose a MoE model** (30B-A3B, gpt-oss-120B) — CPU/partial-offload MoE is the best speed-per-byte outside the GPU.
3. **Dense 70B on CPU** is a batch-job tier (2–4 t/s), not a chat tier.
4. **RAM ≥ file size + 16 GB headroom** for CPU/partial-offload builds; `--mlock` to be safe.
5. **Check the offload log** (`offloaded X/Y layers`) — silent partial offload is the #1 "why is this slow" cause.

## Sources

- llama.cpp docs: GPU offload, `--n-gpu-layers`, `--mlock`/`--mmap` (see [docs/references.md](../docs/references.md))
- Community benchmarks: llama.cpp discussion #4167 (Apple), kunalganglani.com/llm-benchmarks (CPU/GPU tok/s database)
- [vram-formula.md](vram-formula.md) for the VRAM side of the equation

## My experiments

- (fill in: e.g. "Qwen3-30B-A3B Q4 on my Ryzen 9 7950X (DDR5-6000 dual): 42 t/s decode, 900 t/s prefill — beats my 13B dense on the same box")
