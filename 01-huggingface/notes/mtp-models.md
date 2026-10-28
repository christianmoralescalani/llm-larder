# MTP (Multi-Token Prediction) models on the Hub

**Summary.** MTP models carry a small extra "head" trained to predict the *next* token(s) — at inference it acts as a nearly-free draft model for speculative decoding (theory: `00-foundations/notes/multi-token-prediction.md`). On the Hub, MTP shows up in **three different ways**, and that's the trap: (1) a field in `config.json`, (2) tensors embedded in the main weights, or (3) a **separate sidecar/assistant file** you must download too. Know which one you're looking at before you run the model.

## The three MTP layouts

| Layout | Where it lives | Examples (verified 2026-10-27) |
|---|---|---|
| **Config flag + embedded tensors** | `config.json` has `num_nextn_predict_layers: 1`; MTP tensors sit in the main safetensors/GGUF | DeepSeek-V3 / R1 (`num_nextn_predict_layers: 1`), Qwen3-Next-80B-A3B (MTP in official README; tensors in weights, no config field), XiaomiMiMo-7B (vLLM docs example) |
| **Separate sidecar file (GGUF)** | `mtp-<model>-<quant>.gguf` next to the main GGUF | produced by `convert_hf_to_gguf.py --mtp`; `llama-server -hf` pulls it automatically as a "sidecar" |
| **Separate assistant checkpoint (repo)** | its own HF repo, e.g. `gg-hf-am/gemma-4-E2B-it-assistant` | **Gemma 4** family (E2B, E4B, 12B, 26B-A4B, 31B IT) — per vLLM docs |

Counter-example: **Kimi-K2** — the paper mentions MTP training, but the public checkpoint has `num_nextn_predict_layers: 0` (verified in its `config.json`). No MTP head to use. Always check the actual config/tensors, not the paper.

## How to check if a model has MTP

1. **config.json**: `curl -s https://huggingface.co/<repo>/resolve/main/config.json | jq '.num_nextn_predict_layers'` (DeepSeek family).
2. **Tensor names**: in safetensors look for `mtp.*` tensors (check `model.safetensors.index.json` — no download needed); in GGUF look for `blk.N.mtp.*` or `mtp.*` tensors (see `cheatsheets/inspect-weights.md`).
3. **Sidecar files**: `hf models ls <repo> -R | grep -E 'mtp-|assistant'`.
4. **Model card**: "Multi-Token Prediction" in the highlights (Qwen3-Next does this).

## Which models have MTP (as of 2026-10-27)

| Model | MTP layout | Notes |
|---|---|---|
| DeepSeek-V3 / R1 | embedded (1 layer) | the original; runs as self-speculative draft |
| Qwen3-Next-80B-A3B | embedded | official README: "Multi-Token Prediction (MTP): … accelerates inference" |
| Gemma 4 (E2B…31B IT) | separate **assistant checkpoint** | vLLM: `--speculative-config '{"method":"mtp","model":"gg-hf-am/gemma-4-E2B-it-assistant",...}'`; llama.cpp: `draft-mtp` |
| XiaomiMiMo-7B | embedded | vLLM docs example |
| Mellum2.1 | embedded | llama.cpp support added (PR #30612) |
| Kimi-K2 | **none** in public weights | `num_nextn_predict_layers: 0` |

## Running MTP models

**llama.cpp** (current master, verified 2026-10-27):

```bash
# MTP module embedded in the GGUF (DeepSeek, Qwen3-Next, …)
llama-server -m DeepSeek-R1-Q4_K_M.gguf --spec-type draft-mtp

# or let the server pull model + mtp sidecar from the Hub
llama-server -hf <org>/<model>-GGUF:Q4_K_M

# tune the draft depth (default 3)
llama-server -m model.gguf --spec-type draft-mtp --spec-draft-n-max 3
```

`--spec-type` accepts `none,draft-simple,draft-eagle3,draft-mtp,draft-dflash,draft-dspark,ngram-*` (comma list). Watch acceptance in the Prometheus metrics: `llamacpp:spec_decode_num_accepted_tokens_total` vs `…_num_draft_tokens_total`.

**vLLM**:

```bash
# embedded MTP
vllm serve XiaomiMiMo/MiMo-7B-Base \
  --speculative-config '{"method":"mtp","num_speculative_tokens":1}'

# Gemma 4: separate assistant checkpoint
vllm serve google/gemma-4-E2B-it \
  --speculative-config '{"method":"mtp","model":"gg-hf-am/gemma-4-E2B-it-assistant","num_speculative_tokens":1}'
```

`num_speculative_tokens: 1` is the safe default. vLLM only supports MTP for model families it implements — if yours isn't listed, use EAGLE or a draft model instead.

**Producing your own GGUF**:

```bash
# main model without the MTP tensors + a separate mtp- sidecar file
python convert_hf_to_gguf.py <hf-dir> --outdir out --no-nextn
python convert_hf_to_gguf.py <hf-dir> --outdir out --mtp   # writes mtp-<name>-<quant>.gguf
```

## What to expect

- MTP drafts are cheap (small module), so even a modest acceptance rate speeds up decode — typically the biggest win on bandwidth-bound (consumer GPU) setups.
- Acceptance varies by task: code/structured text drafts well, creative text less so.
- It uses a bit of extra VRAM (draft KV cache, `--spec-draft-n-ubatch` to cap).
- Measure before believing: `llama-bench` with and without `--spec-type draft-mtp` on your hardware.

## My experiments

- (fill in: e.g. "DeepSeek-R1-Distill-Qwen-7B Q4_K_M with draft-mtp: 21 → 34 tok/s decode on RTX 4060, acceptance ~1.6 tokens/step on code prompts")
