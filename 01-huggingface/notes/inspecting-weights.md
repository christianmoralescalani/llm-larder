# Seeing the weights — inspecting models before and after download

**Summary.** "See weights" in practice means four questions: **what files and how big** (VRAM budget), **what does the config say** (architecture, context, MoE, MTP), **what precision are the actual tensors in** (GGUF quants per tensor, safetensors dtypes), and **is the download intact** (checksums). All of it is doable without loading the model — with the `hf` CLI, the Hub API, and two small Python packages (`gguf`, `safetensors`).

## 1. List files + sizes (before downloading)

```bash
# CLI: file tree with human sizes
hf models ls unsloth/Qwen2.5-VL-7B-Instruct-GGUF -R -h
hf models ls unsloth/Qwen2.5-VL-7B-Instruct-GGUF --tree -h

# API: names + sizes (blobs=true)
curl -s "https://huggingface.co/api/models/unsloth/Qwen2.5-VL-7B-Instruct-GGUF?blobs=true" \
  | jq -r '.siblings[] | select(.rfilename | endswith(".gguf")) | "\(.size // 0)\t\(.rfilename)"'

# dry-run: what would be downloaded + total size
hf download Qwen/Qwen3-8B-GGUF --dry-run
```

Then estimate VRAM: `weights GB + KV cache + ~1 GB overhead` (formulas in `02-hardware-and-vram/`).

## 2. Read `config.json` (safetensors repos)

```bash
curl -s https://huggingface.co/Qwen/Qwen3-8B/resolve/main/config.json | jq .
```

| Field | Tells you |
|---|---|
| `architectures` / `model_type` | which runtime code path (e.g. `Qwen3ForCausalLM`) |
| `max_position_embeddings` | context length |
| `num_hidden_layers`, `hidden_size` | depth/width → VRAM math |
| `num_attention_heads`, `num_key_value_heads` | GQA ratio → KV cache size |
| `sliding_window` | local attention window (hybrid models) |
| `torch_dtype` | native precision (`bfloat16`, `float16`) |
| `quantization_config` | GPTQ/AWQ/FP8 if present |
| `num_experts`, `num_experts_per_tok` | MoE: total vs active experts |
| `num_nextn_predict_layers` | **MTP** (>0 = has MTP head; DeepSeek family) |

Multimodal configs nest the LLM under `text_config` (e.g. Gemma 4, Qwen3-VL) — check both levels.

## 3. Parameter count

```bash
# safetensors repos: exact count from the API
curl -s "https://huggingface.co/api/models/Qwen/Qwen3-8B?expand[]=safetensors" | jq '.safetensors'
# → {"parameters": {"BF16": 8190735360}, "total": 8190735360}   (≈ 8.19 B)
```

GGUF repos don't expose this via the API — estimate `file size ÷ bytes/weight` (table in `00-foundations/cheatsheets/quantization-levels.md`) or count tensors (below).

## 4. Inspect a GGUF (after download)

`gguf` Python package (v0.19, also vendored as `gguf-py` in the llama.cpp repo):

```python
from gguf import GGUFReader
from collections import Counter

r = GGUFReader("Qwen3-8B-Q4_K_M.gguf")

# metadata — v0.19 reader API: get_field(key).contents()
# (no .architecture / .get_chat_template() / [] access in current gguf)
arch = r.get_field("general.architecture")
print(arch.contents() if arch else "<missing>")             # e.g. "qwen3"

tmpl = r.get_field("tokenizer.chat_template")
print(tmpl.contents()[:200] if tmpl else "<no template>")   # jinja, if embedded

# per-tensor quantization: which tensors are at which precision
c = Counter(t.tensor_type.name for t in r.tensors)
for t, n in c.most_common():
    print(f"{t:12s} {n}")

# MTP check: any mtp tensors?
print([t.name for t in r.tensors if "mtp" in t.name][:10])
```

What to look for:
- **Architecture string** matches what you think you downloaded.
- **Chat template present** (if not, the runtime needs `--jinja`/`--chat-template` — see `06-sampling-and-prompting`).
- **Sensitive tensors at high precision** (attention output, embeddings) in `_M`/`_L` quants — that's what makes Q4_K_M good.
- **MTP tensors** (`blk.N.mtp.*` / `mtp.*`) if you expect MTP.

## 5. Inspect safetensors (after download)

```python
from safetensors import safe_open

with safe_open("model-00001-of-00004.safetensors", framework="pt") as f:
    for k in list(f.keys())[:5]:
        t = f.get_slice(k)
        print(k, t.get_shape(), t.get_dtype())
```

For large sharded repos, read `model.safetensors.index.json` first — it maps every tensor to its shard, so you can check for `mtp.*` / vision-tower tensors **without downloading anything**:

```bash
curl -s https://huggingface.co/<repo>/resolve/main/model.safetensors.index.json \
  | jq '[.weight_map | keys[] | select(test("mtp|vision"))] | length'
```

## 6. Verify integrity

```bash
# checksums against the Hub (cache or --local-dir)
hf cache verify Qwen/Qwen3-8B-GGUF
hf cache verify Qwen/Qwen3-8B-GGUF --local-dir /path/to/repo

# housekeeping
hf cache ls --filter "size>10g"
hf cache rm model/<org>/<model> -y
hf cache prune                 # detached revisions + .incomplete files
```

Manual: `sha256sum file.gguf` and compare against the repo's commit page if you're paranoid (CDN corruption is rare; interrupted downloads are the usual suspect — `hf download` resumes, plain `wget` doesn't).

## My experiments

- (fill in: e.g. "gguf reader on bartowski Qwen3-8B Q4_K_M: 92% of tensors Q4_K, attn_v + token_emb at Q6_K/Q8_0 — matches the _M mix")
