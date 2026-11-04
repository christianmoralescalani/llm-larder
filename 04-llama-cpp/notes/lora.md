# LoRA adapters

**Summary.** LoRA adapters are small GGUF delta-weights that modify a base model at load time — llama.cpp applies them in memory, so the base GGUF stays untouched. Three ways to use them: **load at startup** (`--lora` on `llama-server`/`llama-cli`), **switch at runtime** (`POST /lora-adapters` on a server started with `--lora-init-without-apply`), or **merge permanently** (`llama-export-lora` bakes the adapter into a new GGUF). Multiple adapters can be stacked, each with its own scale.

## Loading at startup

```bash
# one adapter (server)
docker run -d --name llm -p 8080:8080 --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /m/base-model.gguf --lora /m/adapter.gguf \
  --host 0.0.0.0 --port 8080

# several adapters at once (comma-separated), or with explicit scales
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  --model /m/base-model.gguf \
  --lora-scaled /m/style-a.gguf:0.7,/m/style-b.gguf:0.3
```

Flags (server and CLI):

| Flag | Notes |
|---|---|
| `--lora FNAME[,FNAME…]` | adapter(s) applied at load |
| `--lora-scaled FNAME:SCALE,…` | adapters with explicit scaling |
| `--lora-init-without-apply` | load adapters but apply none — switch via API later |

## Switching at runtime (server)

Start with the adapters loaded but unapplied, then list/apply over HTTP:

```bash
# server start
... --lora /m/a.gguf,/m/b.gguf --lora-init-without-apply

# what's available
curl -s http://localhost:8080/lora-adapters

# apply a set (empty list = base model only)
curl -s -X POST http://localhost:8080/lora-adapters -d '{"lora": ["/m/a.gguf"]}'
```

This is how you run one server that serves "base", "style A", and "A+B" without reloading weights.

## Merging permanently

`llama-export-lora` (in the `full` image) bakes adapter(s) into a new GGUF — useful for distribution or when you want zero runtime overhead:

```bash
docker run --rm -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full \
  llama-export-lora -m /m/base-model.gguf -o /m/base-merged.gguf \
  --lora /m/adapter.gguf

# stacked with scales
docker run --rm -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full \
  llama-export-lora -m /m/base-model.gguf -o /m/base-merged.gguf \
  --lora-scaled /m/a.gguf:0.5,/m/b.gguf:0.5
```

## Where adapters come from

- Hugging Face: GGUF-format LoRAs (search `lora` + model name; e.g. ggml-org or community repos). They must match the **base model's architecture and tokenizer** — a Llama-3.1 adapter won't load on Llama-3.2.
- Train your own with any LoRA trainer (Axolotl, LLaMA-Factory, unsloth…), then convert the safetensors checkpoint to GGUF with `convert_hf_to_gguf.py` (LoRA mode).

## Gotchas

1. **Base must match exactly** — architecture, tokenizer, and (for some) context; mismatch = load error or silent garbage.
2. **Scales are per-adapter** — `--lora-scaled a.gguf:0.5,b.gguf:0.5`; stacking at 1.0 each is not the same as a single 1.0 adapter.
3. **`--lora-init-without-apply` + `POST /lora-adapters`** is the only runtime path — plain `--lora` is fixed for the process lifetime.
4. **Merged output is F16-ish by default** (`ggml-lora-merged-f16.gguf`) — re-quantize the merged file if you need the smaller size back.
5. **MoE + LoRA** — adapters trained on the dense base may misbehave on MoE variants; verify with a quick `llama-cli` chat.

## Sources

- [export-lora README](https://github.com/ggml-org/llama.cpp/blob/master/tools/export-lora/README.md)
- [Server README — LoRA endpoints](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)

## My experiments

- (fill in: e.g. "loaded adapter X on base Y via `--lora` — style change visible in `llama-cli`; `POST /lora-adapters` switch took ~Z s without reload; merged file via `llama-export-lora` was 2.1% larger than base")
