# llama-cli — interactive use

**Summary.** `llama-cli` is the terminal chat REPL (also available as `llama cli` on the unified `llama` binary, v0.8.0+). It loads a GGUF, applies the model's chat template (Jinja by default), and gives you a multi-turn conversation with per-turn timings. It's the fastest way to test a model, check a chat template, or try a quant before serving it. Same model/memory/sampling flags as `llama-server` (see `cheatsheets/llama-server-flags.md`); the CLI-specific ones are below.

## Quick start (Docker)

```bash
# interactive chat, GPU, from a local GGUF (light image has llama-cli)
docker run --rm --gpus all -v ~/Downloads/models:/models \
  ghcr.io/ggml-org/llama.cpp:light-cuda \
  --model /models/model.gguf --n-gpu-layers -1 --ctx-size 8192

# straight from Hugging Face
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF

# one-shot (non-interactive): prompt in, answer out, exit
docker run --rm ghcr.io/ggml-org/llama.cpp:light \
  -hf ggml-org/Qwen3.5-0.8B-GGUF \
  -p "Explain KV cache in two sentences" --single-turn
```

Inside the REPL: type and Enter to send; `\` at line end continues a multi-line input (or use `--multiline-input`); `Ctrl-D` / `Ctrl-C` exits; `--reverse-prompt` halts generation when the model emits that string (classic multi-turn trick).

## CLI-specific flags

| Flag | Notes |
|---|---|
| `-p, --prompt` | first-turn prompt (with `-st`, that's the only turn) |
| `-f, --file` / `-bf, --binary-file` | prompt from a file |
| `-sys, --system-prompt` / `-sysf` | system message (used by the chat template) |
| `-st, --single-turn` | one turn then exit (batch/one-shot mode) |
| `-r, --reverse-prompt` | stop generation when this string appears (return to input) |
| `-mli, --multiline-input` | paste multi-line without trailing `\` |
| `--server-base URL` | talk to a running `llama-server` instead of loading a model |
| `--show-timings` (default on) / `--no-show-timings` | per-turn prompt/decode tok/s |
| `--verbose-prompt` | dump the fully rendered prompt (chat template output) — great for debugging templates |
| `--display-prompt` (default on) | print the prompt before generating |
| `-co, --color` | colorize prompt vs generation (`auto` default) |
| `--simple-io` | basic IO for subprocesses/limited consoles |
| `--image, --audio, --video FILE` | media input for multimodal models (comma-separated for multiple) |
| `-mm, --mmproj FILE` | projector file (auto with `-hf`) |
| `--log-prompts-dir PATH` | dump rendered prompts to disk |

## Chat template & reasoning

- `--jinja` (default **on**): render the model's Jinja template from GGUF metadata. With a template, `-p` becomes a *user message* and `-sys` a *system message* — you don't hand-format chat markers.
- `--chat-template NAME` / `--chat-template-file FILE`: override (built-ins: `llama3`, `chatml`, `deepseek3`, `kimi-k2`, …).
- `--reasoning [on|off|auto]` (default `auto`): thinking on/off; `--reasoning-effort`, `--reasoning-budget N` (token budget) map to template variables.
- `--skip-chat-parsing`: everything lands in content (no reasoning/tool-call splitting).
- `--verbose-prompt` is the debugging tool: it shows exactly what the template produced, so you can see system prompt placement, thinking tags, tool-call blocks.

## Sampling & constraints

Same as the server: `--temp`, `--top-k`, `--top-p`, `--min-p`, `--repeat-penalty`, `--seed`, `--samplers`, plus:

- `--grammar GRAMMAR` / `--grammar-file`: BNF constraint (examples in `grammars/` upstream).
- `-j, --json-schema` / `-jf`: JSON-schema constrained output.
- `-l, --logit-bias ID±BIAS`: nudge a token's likelihood.
- `--ignore-eos`: keep generating past EOS.

## Memory & performance (same as server)

`--ctx-size`, `--n-gpu-layers`/`-ngl`, `--flash-attn`, `--cache-type-k/v`, `--batch-size`/`--ubatch-size`, `--threads`/`--threads-batch`, `--numa`, `--load-mode` (`mlock` to pin in RAM), `--spec-type` + draft flags for speculative decoding. The per-turn timing line reports `prompt eval` and `eval` tok/s — use it to sanity-check a quant or flag change before serving.

## Gotchas

1. **`-p` + template = user message** — with Jinja on, `-p "..."` is wrapped as a user turn; don't pre-format chat markers yourself.
2. **`--single-turn` + `-p`** is the one-shot mode; without `-st` the CLI stays interactive after the first turn.
3. **`--ctx-size` default is the model's** (often small, e.g. 4096–8192) — set it explicitly for long chats.
4. **`--reverse-prompt` is positional** — the string must match the model's output exactly (e.g. the user marker), or generation never halts.
5. **Timings lie under load** — `eval` tok/s drops if the CPU is busy elsewhere; compare runs back-to-back.
6. **`--server-base`** turns llama-cli into a thin client for a running server — handy for trying different sampling flags without reloading the model.

## Sources

- [CLI README (auto-generated flag reference)](https://github.com/ggml-org/llama.cpp/blob/master/tools/cli/README.md)
- [llama.cpp README (unified `llama` binary)](https://github.com/ggml-org/llama.cpp/blob/master/README.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B Q4_K_XL via `llama cli` — `--verbose-prompt` shows the Jinja template puts the system prompt inside the user turn; `--reasoning on` adds thinking tokens, `--reasoning-budget 2048` caps them; one-shot `-p … --single-turn` returns in X s with Y tok/s eval")
