# Cheatsheet — llama-cli flags

**TL;DR.** `llama-cli` = terminal chat REPL (also `llama cli` on the unified binary). Interactive by default; `-p "…" --single-turn` for one-shot. `-sys` = system message, `-r` = reverse prompt (halt string), `--verbose-prompt` = show the rendered chat template. Everything else is shared with `llama-server` — see `llama-server-flags.md`. Verified against llama.cpp master (v0.8.0, 2026-10-28), 2026-11-03.

## CLI-specific

| Flag | Default | Notes |
|---|---|---|
| `-p, --prompt` | — | first-turn prompt (user message under a template) |
| `-f, --file` / `-bf, --binary-file` | — | prompt from file |
| `-sys, --system-prompt` / `-sysf` | — | system message |
| `-st, --single-turn` | off | one turn then exit (one-shot) |
| `-r, --reverse-prompt` | — | halt generation at this string |
| `-mli, --multiline-input` | off | paste multi-line without `\` |
| `--server-base URL` | — | use a running server instead of loading |
| `--show-timings` | on | per-turn tok/s (`--no-show-timings` off) |
| `--verbose-prompt` | off | print rendered prompt (template debugging) |
| `--display-prompt` | on | print prompt before generating |
| `-co, --color` | auto | colorize prompt vs generation |
| `--simple-io` | off | basic IO for subprocesses |
| `--image, --audio, --video FILE` | — | media input (comma-separated) |
| `-mm, --mmproj` | — | projector (auto with `-hf`) |
| `--mmproj-offload` | on | GPU encoder | `MTMD_BACKEND_DEVICE` |
| `--image-min-tokens` / `--image-max-tokens` | model | dynamic-resolution bounds |
| `--log-prompts-dir` | off | dump rendered prompts |
| `-ctxcp, --ctx-checkpoints` | 32 | context checkpoints |
| `-cram, --cache-ram` | 8192 MiB | prompt-cache size |
| `--context-shift` | off | infinite-gen sliding context |
| `--warmup` | on | empty warmup run |

## The ones you'll actually use

| Flag | Default | Notes |
|---|---|---|
| `-m` / `-hf` | — | model path / HF repo[:quant] |
| `-c, --ctx-size` | model | set it explicitly (model default is often small) |
| `-n, --n-predict` | -1 (∞) | cap tokens |
| `-ngl, --n-gpu-layers` | auto | `-1` = all layers on GPU |
| `-fa, --flash-attn` | auto | `on` + q4_0 KV for VRAM |
| `-ctk` / `-ctv` | f16 | KV dtypes |
| `--temp` / `--top-k` / `--top-p` / `--min-p` | 0.8 / 40 / 0.95 / 0.05 | sampling |
| `--repeat-penalty` | 1.0 (off) | |
| `-s, --seed` | -1 | reproducibility |
| `--jinja` | **on** | chat template engine |
| `--reasoning` | auto | `on`/`off` for thinking models |
| `--reasoning-effort` / `--reasoning-budget` | default / -1 | thinking controls |
| `--grammar` / `-j, --json-schema` | — | constrained output |
| `-l, --logit-bias` | — | `ID±BIAS` |
| `--ignore-eos` | off | |
| `-t, --threads` / `-tb` | auto | CPU threads |
| `--lora` / `--lora-scaled` | — | adapters |
| `--spec-type` + `--spec-draft-*` | none | speculative decoding (same as server) |
| `--numa` | off | multi-socket CPUs |
| `--load-mode` | auto | `mlock` to pin in RAM |

## One-liners

```bash
# interactive chat (GPU)
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  -m /m/model.gguf -ngl -1 -c 8192

# one-shot
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF \
  -p "Explain KV cache in two sentences" --single-turn

# with system prompt + JSON output
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF \
  -sys "You answer with JSON only" -p "Population of France" \
  -j '{"type":"object","properties":{"population":{"type":"integer"}}}' --single-turn

# vision
docker run --rm --gpus all -v ~/Downloads:/d \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  -hf ggml-org/gemma-3-4b-it-GGUF --image /d/photo.jpg \
  -p "Describe this image" --single-turn

# debug the chat template
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF -p "hi" --verbose-prompt --single-turn

# as a client of a running server
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  --server-base http://host.docker.internal:8080 -p "hello" --single-turn
```
