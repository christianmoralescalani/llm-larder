# Cheatsheet — Reading GGUF filenames & picking a quant

**TL;DR.** `<base-name>-<QUANT>.gguf` — the quant is in the name. Sharded files (`-00001-of-00004`) must **all** be present. `mmproj-*` = vision projector, `mtp-*` = MTP draft sidecar. Default pick: **Q4_K_M**; go up if VRAM allows, down only when forced.

## Filename anatomy

```text
Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf
└──────── base ────────┘└quant┘

Qwen3-Next-80B-A3B-Instruct-00001-of-00004.gguf   sharded: 4 files, all required
Qwen2.5-VL-7B-Instruct-UD-IQ2_M.gguf             UD = unsloth dynamic quant
mmproj-Qwen3VL-8B-Instruct-Q8_0.gguf             vision projector (must start with "mmproj")
mtp-Qwen3-Next-80B-A3B-Instruct-Q4_K_M.gguf      MTP draft sidecar
```

## Quant picker (bytes/weight from `00-foundations/cheatsheets/quantization-levels.md`)

| Situation | Pick |
|---|---|
| Default, 3B+ model | **Q4_K_M** (~0.61 B/w) |
| Have headroom, want max quality | Q5_K_M → Q6_K → Q8_0 |
| Small model (≤3B) | prefer Q6_K/Q8_0 (less redundancy to absorb error) |
| Tight space, big MoE | IQ4_XS / UD-IQ2_M / UD-Q4_K_XL (imatrix/dynamic) |
| Below Q4 | only when nothing else fits — visible quality loss |
| Testing / reference | F16 or BF16 |

Quick size: `params(B) × B/weight` → 8B Q4_K_M ≈ 5.0 GB, 8B Q8_0 ≈ 8.7 GB, 70B Q4_K_M ≈ 43 GB.

## Sidecar files

| Prefix | What | Notes |
|---|---|---|
| `mmproj-*` | vision tower + projector | must match the exact base model; F16/BF16/F32 or Q8_0; extra VRAM |
| `mtp-*` | MTP draft head | for `--spec-type draft-mtp`; `llama-server -hf` pulls it automatically |
| (none) | plain LLM | — |

Vision model = **two files** (LLM + mmproj). MTP model with sidecar layout = **two files** (LLM + mtp-).

## Sharding

- `-00001-of-NNNNN.gguf`: large models (MoE, 32B+) are split; download **all** shards.
- Shards share one metadata header; any missing shard = load failure.
- `hf download <repo> --include "*Q4_K_M*"` grabs the right set.

## Sanity checks before running

1. Quant in name matches what you think you downloaded.
2. All shards present (count = `of-NNNNN`).
3. Vision: mmproj present and from the same repo/base model.
4. MTP: `mtp-` sidecar present (or embedded — check with `gguf` reader).
5. Size ≈ `params × B/weight` (±10%) — a 2× deviation means wrong file.

## Verified on

- 2026-10-27, file listings from `unsloth/Qwen2.5-VL-7B-Instruct-GGUF` (mmproj-F16/BF16/F32), `Qwen/Qwen3-VL-8B-Instruct-GGUF` (mmproj-…-F16/Q8_0), `Qwen/Qwen3-Next-80B-A3B-Instruct-GGUF` (sharded + quants), `unsloth/Kimi-K2-Instruct-GGUF` (per-quant subfolders).
