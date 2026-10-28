# Cheatsheet — Seeing the weights (inspect models)

**TL;DR.** Four steps: **list files/sizes** → **read `config.json`** → **check tensor precisions** (GGUF or safetensors) → **verify checksums**. Nothing requires loading the model.

## 1. Files + sizes (before download)

```bash
hf models ls <org>/<model> -R -h
curl -s "https://huggingface.co/api/models/<org>/<model>?blobs=true" \
  | jq -r '.siblings[] | "\(.size // 0)\t\(.rfilename)"'
hf download <org>/<model> --dry-run
```

VRAM check: `weights GB + KV cache + ~1 GB overhead` (see `02-hardware-and-vram/`).

## 2. `config.json` (safetensors repos)

```bash
curl -s https://huggingface.co/<repo>/resolve/main/config.json | jq .
```

| Field | Meaning |
|---|---|
| `max_position_embeddings` | context length |
| `num_hidden_layers` / `hidden_size` | depth / width |
| `num_key_value_heads` | GQA ratio → KV cache size |
| `sliding_window` | hybrid/local attention window |
| `torch_dtype` | native precision |
| `quantization_config` | GPTQ/AWQ/FP8 (if present) |
| `num_experts` / `num_experts_per_tok` | MoE total / active |
| `num_nextn_predict_layers` | **MTP** (>0 = MTP head) |

Multimodal (Gemma 4, Qwen3-VL): LLM fields live under `text_config`.

## 3. Parameter count

```bash
curl -s "https://huggingface.co/api/models/Qwen/Qwen3-8B?expand[]=safetensors" | jq '.safetensors.total'
# 8190735360  (≈ 8.19 B)
```

GGUF repos: no API count → `file size ÷ bytes/weight` (table in `00-foundations/cheatsheets/quantization-levels.md`).

## 4. GGUF tensor inspection

```python
# pip install gguf
from gguf import GGUFReader
from collections import Counter

r = GGUFReader("model-Q4_K_M.gguf")
arch = r.get_field("general.architecture")
print(arch.contents() if arch else "<missing>")   # e.g. "qwen3"
tmpl = r.get_field("tokenizer.chat_template")
print(bool(tmpl))                                 # chat template embedded?

c = Counter(t.tensor_type.name for t in r.tensors)
for t, n in c.most_common():
    print(f"{t:12s} {n}")             # per-tensor quant mix

print([t.name for t in r.tensors if "mtp" in t.name][:5])   # MTP check
```

## 5. safetensors inspection

```python
# pip install safetensors
from safetensors import safe_open
with safe_open("model-00001-of-00004.safetensors", framework="pt") as f:
    for k in list(f.keys())[:5]:
        s = f.get_slice(k)
        print(k, s.get_shape(), s.get_dtype())
```

No download needed to check tensor names of a sharded repo:

```bash
curl -s https://huggingface.co/<repo>/resolve/main/model.safetensors.index.json \
  | jq '[.weight_map | keys[] | select(test("mtp|vision"))]'
```

## 6. Integrity

```bash
hf cache verify <repo>                    # checksums vs Hub
hf cache verify <repo> --local-dir /path  # plain folder
hf cache prune                            # remove .incomplete / detached
sha256sum file.gguf                       # manual
```

## Verified on

- 2026-10-27: API calls executed live (Qwen3-8B = 8,190,735,360 params); `gguf` PyPI v0.19.0 (reader API `get_field(...).contents()` checked against the v0.19.0 wheel source); `hf cache verify` from current `hf` CLI docs.
