# Quantizing your own models

**Summary.** Two phases: (1) `convert_hf_to_gguf.py` (repo root, Python) turns a Hugging Face repo into a high-precision GGUF (BF16/F16); (2) `llama-quantize` (C++ binary) shrinks it to the target quant (`Q4_K_M`, `Q5_K_M`, `IQ4_XS`, …). An **imatrix** (importance matrix, computed by `llama-imatrix` on a text corpus) is the single biggest quality lever for ≤5-bit quants. All of this runs in the `full` Docker image — no host toolchain needed. Quality is measured with `llama-perplexity` (ppl) and `llama-bench` (speed); see 02-hardware-and-vram for size math.

## Pipeline

```
HF repo (safetensors)
   │  python convert_hf_to_gguf.py  (Python, transformers)
   ▼
model-bf16.gguf  (high precision, big)
   │  llama-quantize  (C++, + optional imatrix)
   ▼
model-Q4_K_M.gguf  (what you actually run)
```

For multimodal models, the vision/audio encoder + projector is a **separate** GGUF (`mmproj-*.gguf`), usually kept at Q8_0/BF16 because its quality directly affects generation.

## Step 1 — HF → GGUF (Docker, `full` image)

```bash
# remote repo, BF16 output (auto-detects the arch)
docker run --rm -v ~/Downloads/models:/out \
  ghcr.io/ggml-org/llama.cpp:full \
  python convert_hf_to_gguf.py --remote google/gemma-3-4b-it --outfile /out/model-bf16.gguf --outtype bf16

# local weights you already downloaded (pass the dir, no --remote)
docker run --rm -v ~/Downloads/hf-model:/src -v ~/Downloads/models:/out \
  ghcr.io/ggml-org/llama.cpp:full \
  python convert_hf_to_gguf.py /src --outfile /out/model-bf16.gguf --outtype f16

# multimodal projector (separate file, keep it high-precision)
docker run --rm -v ~/Downloads/models:/out \
  ghcr.io/ggml-org/llama.cpp:full \
  python convert_hf_to_gguf.py --mmproj --remote google/gemma-3-4b-it \
  --outfile /out/mmproj-Q8_0.gguf --outtype q8_0
```

Notes:
- `--outtype auto` (default) works for the usual 16-bit-distributed models; use `bf16`/`f16` explicitly for control.
- The image pins transformers 4 for compatibility; if a new model needs transformers 5, `pip install -U transformers` inside the container (or a derived image).
- `--remote` pulls via the HF hub; `HF_TOKEN` env var handles gated repos.

## Step 2 — Quantize (Docker, `full` image)

```bash
# the standard: Q4_K_M from the BF16 file
docker run --rm -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full \
  llama-quantize /m/model-bf16.gguf /m/model-Q4_K_M.gguf Q4_K_M

# with an imatrix (recommended for Q4 and below)
docker run --rm -v ~/Downloads/models:/m -v ~/calibration:/cal \
  ghcr.io/ggml-org/llama.cpp:full \
  llama-quantize --imatrix /cal/imatrix.gguf /m/model-bf16.gguf /m/model-Q4_K_M.gguf Q4_K_M
```

Key options:

| Option | Effect |
|---|---|
| `--imatrix FILE` | importance matrix guides per-tensor quant choice (big quality win at ≤5-bit) |
| `--include-weights NAME` / `--exclude-weights NAME` | restrict the imatrix to / from specific tensors (can't mix) |
| `--tensor-type NAME=TYPE` | force a type for specific tensors, regex OK, repeatable (e.g. `attn_v=q5_k`) |
| `--output-tensor-type` / `--token-embedding-type` | keep `output.weight` / embeddings at a chosen type (often Q8_0 for quality) |
| `--leave-output-tensor` | don't quantize `output.weight` at all (size ↑, quality ↑, esp. when requantizing) |
| `--allow-requantize` | quantize from an already-quantized file (quality loss — avoid when you have the BF16) |
| `--pure` | no k-quant mixtures: every tensor the same type |
| `--keep-split` | keep the input's shard layout instead of one file |
| `--prune-layers N,M,…` | remove layers (research/experiments) |

## Choosing the quant

Size math lives in `02-hardware-and-vram/notes/quantization-math.md`; the short version:

| Target | Typical choice | When |
|---|---|---|
| best quality that fits | `Q6_K` / `Q8_0` | plenty of VRAM; Q8_0 ≈ lossless for most uses |
| the default sweet spot | `Q4_K_M` | what everyone ships; ~4.9 bpw |
| tight VRAM | `Q4_K_S`, `IQ4_XS`, `Q3_K_M` | imatrix strongly recommended at Q3/IQ4 |
| extreme squeeze | `Q2_K`, `IQQ*` | noticeable degradation; measure, don't guess |

Rules of thumb:
- **Q4_K_M + imatrix** is the quality/size default for 7B–32B models.
- Keep `output.weight` and token embeddings at higher precision (`--output-tensor-type q8_0`) if you have headroom.
- MoE models: `--cpu-moe`/`--n-cpu-moe` at runtime is often the better lever than a lower quant.

## Imatrix (importance matrix)

`llama-imatrix` runs a text corpus through the model and records which weights matter, so `llama-quantize` can spend bits where they count.

```bash
# 1. get calibration text (wikitext-style raw text; 100–500 MB is plenty)
# 2. compute the imatrix (GPU offload helps a lot)
docker run --rm --gpus all -v ~/Downloads/models:/m -v ~/calibration:/cal \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-imatrix -m /m/model-bf16.gguf -f /cal/wiki.txt -o /cal/imatrix.gguf --n-gpu-layers -1
```

Notes:
- Output is GGUF by default (`-o imatrix.gguf`); legacy `.dat` if you name it so.
- `--process-output` also collects data for `output.weight`/embeddings (off by default — usually better that way).
- `--nextn` / `--model-draft` collect data for MTP/NextN layers (for models that have them).
- `--in-file` merges imatrices from multiple runs/datasets.
- The imatrix is **model-specific** — don't reuse one across models.

## Verify: perplexity + bench

```bash
# quality: lower ppl = closer to the source model
docker run --rm -v ~/Downloads/models:/m -v ~/calibration:/cal \
  ghcr.io/ggml-org/llama.cpp:full \
  llama-perplexity -m /m/model-Q4_K_M.gguf -f /cal/wiki.txt

# speed on your hardware (prompt processing + token generation, per quant)
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-bench -m /m/model-Q4_K_M.gguf
```

Compare the quant's ppl against the BF16 file's ppl: a small delta (e.g. < 0.1–0.2) means the quant is safe. `llama-bench` output is the canonical way to compare quants/flags — see `notes/llama-server.md` § gotchas for what to hold constant.

## Gotchas

1. **Always quantize from 16-bit**, never from an existing quant (`--allow-requantize` is a last resort; quality compounds).
2. **imatrix is per-model and per-corpus-domain** — a code imatrix helps code quants; a generic wiki imatrix is the safe default.
3. **`output.weight` matters** — quantizing it aggressively (Q2–Q3) is a common cause of "the quant sounds worse than it should"; keep it Q8_0 or unquantized.
4. **mmproj is separate** — forgetting `--mmproj` conversion = no vision; and keep it Q8_0/BF16.
5. **`--outtype` on convert** — converting to `q8_0` directly skips the BF16 file but kills your ability to make multiple quants from one conversion.
6. **GPU for imatrix/quantize** — both are much faster with `--n-gpu-layers -1` on the CUDA image; CPU-only takes a long time on big models.
7. **Hugging Face has a no-setup alternative** — the `ggml-org/gguf-my-repo` Space builds custom quants in the browser (syncs from llama.cpp main every 6 h).

## Sources

- [quantize README](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md)
- [imatrix README](https://github.com/ggml-org/llama.cpp/blob/master/tools/imatrix/README.md)
- [perplexity README](https://github.com/ggml-org/llama.cpp/blob/master/tools/perplexity/README.md)
- [llama-bench README](https://github.com/ggml-org/llama.cpp/blob/master/tools/llama-bench/README.md)

## My experiments

- (fill in: e.g. "quantized X from BF16 → Q4_K_M with imatrix from wiki.raw: ppl 5.87 → 6.01 (+0.14), size 15.2 GiB → 4.6 GiB; llama-bench: prompt 812 t/s, generation 71 t/s; `--output-tensor-type q8_0` cost 0.05 GiB and dropped ppl by 0.04")
