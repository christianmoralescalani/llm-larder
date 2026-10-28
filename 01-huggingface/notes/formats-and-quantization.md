# Model formats & quantizations on the Hub

**Summary.** The same model appears on the Hub in several **formats** (file containers) and **quantizations** (bits per weight). Pick the format by *runtime*: **GGUF** for llama.cpp/Ollama/LM Studio, **safetensors** (fp16/bf16 or GPTQ/AWQ/FP8) for vLLM/Transformers, **MLX** for Apple Silicon. The quantization is encoded in the filename (`Q4_K_M`, `IQ4_XS`, `F16`…). Community quant providers (`ggml-org`, `bartowski`, `unsloth`, `mradermacher`, `lmstudio-community`) each have a style — know them before you pick.

## Which format for which runtime

| Format | Files | Runtime | Notes |
|---|---|---|---|
| **GGUF** | `*.gguf` (often sharded) | llama.cpp, Ollama, LM Studio | Single format for local/consumer hardware; mixed quant per tensor; metadata (chat template, context) inside the file. |
| **safetensors (fp16/bf16)** | `*.safetensors` + `config.json` + tokenizer | Transformers, vLLM, SGLang | Full precision; the "source" most quants are derived from. |
| **GPTQ / AWQ** | `*.safetensors` (int4) | vLLM, Transformers | Calibration-based 4-bit for GPU serving; repo name usually contains `GPTQ`/`AWQ`. |
| **FP8 / NVFP4** | `*.safetensors` | vLLM, SGLang | Hardware-native on Ada/Hopper/Blackwell; near-fp16 quality. |
| **MLX** | `*.safetensors` (mlx layout) | `mlx-lm`, MLX apps | Apple Silicon only. |
| **ONNX** | `*.onnx` | onnxruntime, specialized | Compiled graphs; rare for LLMs. |

**How to tell what a repo is**: look at the files tab (or `hf models ls <repo> -R`). `-GGUF` repo suffix + `gguf` tag = GGUF; `config.json` + `model-00001-of-*.safetensors` = Transformers-style; check the `library` tag.

## Reading a GGUF filename

```text
Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf
└─ base model name ─┘└─ quant ┘
Qwen3-Next-80B-A3B-Instruct-00001-of-00004.gguf   ← sharded (4 files, all required)
Qwen2.5-VL-7B-Instruct-UD-IQ2_M.gguf             ← "UD" = unsloth dynamic quant
mmproj-Qwen3VL-8B-Instruct-F16.gguf              ← vision projector (see mmproj-vision.md)
mtp-Qwen3-Next-80B-A3B-Instruct-Q4_K_M.gguf      ← MTP sidecar draft (see mtp-models.md)
```

Quant quick reference (bytes/weight, details in `00-foundations/cheatsheets/quantization-levels.md`):

| Quant | ~B/weight | When to use |
|---|---|---|
| F16 / BF16 | 2.0 | reference, testing, small models on big GPUs |
| Q8_0 | ~1.06 | max quality that still fits; ≤8B on 24 GB |
| Q6_K | ~0.82 | near-lossless, 14B–32B with headroom |
| Q5_K_M | ~0.72 | quality step above default |
| **Q4_K_M** | ~0.61 | **the default** for 3B+ models |
| Q4_0 / IQ4_XS / UD-IQ2_M | ~0.35–0.57 | tight spaces; IQ/UD use imatrix/dynamic allocation |
| Q3_K_M and lower | <0.5 | only when nothing else fits |

**MoE models** (DeepSeek, Qwen3-Next, Kimi): experts dominate the size, so aggressive expert quantization (IQ/UD quants) costs little quality — that's why a 1T-param MoE runs at ~500 GB in IQ2-ish quants.

## Community quant providers (verified 2026-10-27)

| Provider | Style | Good for |
|---|---|---|
| **ggml-org** | official llama.cpp org; curated pre-quantized GGUFs (incl. **all supported vision models** with mmproj) | `llama-server -hf <repo>` one-liners; vision models |
| **Qwen / meta-llama / deepseek-ai / moonshotai** | official orgs publish their own `-GGUF` repos (often **gated** — login + accept terms) | reference GGUFs straight from the lab |
| **bartowski** | very complete, imatrix quants, abliterated variants, big MoE coverage | when you want a specific quant that exists |
| **unsloth** | fast "dynamic" quants (`UD-*`), vision models, huge MoE models, also fine-tuning | best size/quality squeeze, vision GGUFs |
| **mradermacher** | everything, many quants per model, abliterated | completeness |
| **lmstudio-community** | what LM Studio indexes | LM Studio users |

Search recipe: `https://huggingface.co/models?search=<model name>&filter=gguf&sort=downloads`.

## Non-GGUF quantizations (GPU serving)

For vLLM/Transformers you usually run fp16/bf16 directly, or:

- **GPTQ / AWQ** (int4): search `GPTQ`/`AWQ` in the model name; vLLM picks them up automatically (`--quantization gptq`/`awq`).
- **FP8** (e.g. `*-FP8` repos): near-fp16 quality, half the memory; needs Ada/Hopper/Blackwell.
- **bitsandbytes** int8: on-the-fly, no separate repo needed.

## Gated repos

Official GGUF repos from Qwen, DeepSeek and Moonshot return `401` without a token (verified 2026-10-27). Fix:

1. Open the repo in the browser → accept the terms.
2. `hf auth login` (or `export HF_TOKEN=hf_...`).
3. Download as normal.

## My experiments

- (fill in: e.g. "bartowski Q4_K_M vs unsloth UD-Q4_K_XL on Qwen3-8B: UD file 8% smaller, no visible quality difference on my prompts")
