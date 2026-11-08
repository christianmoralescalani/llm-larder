# Chat templates — practical assembly & debugging

**Summary.** The chat template is the string pattern (Jinja in llama.cpp/HF, Go `text/template` in Ollama) that turns a list of `{role, content}` messages into the exact token sequence the model was trained on. You almost never *write* one — you make sure the right one is *applied*, and you know how to *render and inspect* it when something misbehaves. [00-foundations](../../00-foundations/notes/system-prompts-and-chat-templates.md) covers the concept; this note is the working side: where templates live per runtime, how to see the rendered prompt, what breaks, and how to fix it.

## Where templates live

| Source | Location | Notes |
|---|---|---|
| Hugging Face | `tokenizer_config.json` → `chat_template` (Jinja string) | canonical for the model family |
| GGUF | file metadata (`general.chat_template`) | copied from the HF repo at quant time — **can be stale** |
| Ollama library | embedded in the model manifest | `ollama show <model> --template` to view |
| Ollama Modelfile | `TEMPLATE """…"""` | override for imported GGUFs (Go template syntax) |
| LM Studio / vLLM | from the HF config at load | vLLM: `--chat-template` to override |

**Stale-template trap**: a GGUF quantized before the model's template was revised (e.g. before reasoning support landed) carries the old template. Symptom: model "can't be told to stop thinking", tool calls don't parse. Fix: use a newer quant, or override with `--chat-template-file` / Modelfile `TEMPLATE`.

## Per-runtime behavior

### llama.cpp

- `--jinja` (default **on**): use the Jinja template from GGUF metadata. This is what makes reasoning models, tool calling, and per-model system-prompt quirks work.
- `--chat-template NAME` / `--chat-template-file FILE`: override (built-ins: `llama3`, `chatml`, `mistral-v3`, `deepseek3`, `kimi-k2`, …).
- `--chat-template-kwargs`: extra JSON variables passed into the template (e.g. `{"enable_thinking": false}` for Qwen3).
- `--prefill-assistant` (default on): a trailing assistant message is treated as a prefix to continue, not a new turn.
- `--skip-chat-parsing`: force the pure-content parser (everything in `content`).
- **Render/inspect**:
  - `llama-template` tool (`full` image) — renders a conversation through the GGUF's template, shows the exact prompt.
  - `--log-prompts-dir DIR` on the server — dumps every rendered prompt to disk.
  - `llama-cli --display-prompt` — prints the prompt before generating.

```bash
# see exactly what the server will send to the model
docker run --rm --gpus all -v ~/Downloads/models:/m \
  ghcr.io/ggml-org/llama.cpp:full-cuda \
  llama-template -m /m/model.gguf \
  --system "You are terse." \
  --prompt "Explain vLLM in one sentence."
```

### Ollama

- Library models ship the correct template; you only override with `TEMPLATE` when importing a GGUF Ollama can't infer.
- `SYSTEM """…"""` in the Modelfile = standing system prompt (see 03-ollama).
- **Render/inspect**: `ollama show <model> --template` (the template itself); `ollama run <model> --verbose` (prints the rendered prompt + stats per turn); the API's `prompt` field in `/api/generate` responses shows the assembled prompt.

### vLLM

- Applies the HF `chat_template` automatically; `--chat-template FILE` overrides it.
- `--default-chat-template-kwargs` for server-wide template variables (e.g. disabling Qwen3 thinking).
- **Render/inspect**: `--log-requests` / request logging, or just apply the template yourself with `transformers` to see what the server will do.

### transformers (reference implementation)

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("Qwen/Qwen3-8B")
prompt = tok.apply_chat_template(
    [{"role": "system", "content": "You are terse."},
     {"role": "user", "content": "Explain vLLM in one sentence."}],
    tokenize=False, add_generation_prompt=True,
)
print(prompt)   # the exact string the model sees
```

Use this as the **ground truth** when two runtimes disagree — if llama.cpp's rendered prompt differs from `apply_chat_template`, the GGUF template is stale or a kwarg differs.

## Template variables you'll actually meet

| Variable | Used by | Effect |
|---|---|---|
| `enable_thinking` | Qwen3, Granite, Gemma 4 | on/off the thinking block |
| `thinking_budget` | Qwen3 | token budget for thinking |
| `add_generation_prompt` | all | emit the assistant header so generation starts in the right role |
| `tools` | tool-calling models | render function definitions into the prompt |
| `documents` | Llama 3.2+ | render reference documents (RAG) |
| `reasoning_effort` | effort-aware models | maps to effort levels |

Runtimes inject these automatically (llama.cpp's `--reasoning*` flags, Ollama's `think`, vLLM's `chat_template_kwargs`) — you rarely pass them by hand.

## What breaks, and the fix

| Symptom | Usual cause | Fix |
|---|---|---|
| Answers as "user", mixed roles, or raw markers in output | wrong/missing template | let the runtime apply the shipped template; check `--jinja` is on |
| Output never stops / repeats forever | missing end-of-turn token | correct template; check `stop` sequences as a band-aid |
| Thinking tags leak into `content` | template without reasoning support, or `--jinja` off | newer quant + `--reasoning on` (llama.cpp) / `--reasoning-parser` (vLLM) |
| Tool calls don't parse | template lacks tool support | model's official template (`--chat-template-file`), or a model that supports tools |
| Quality differs between Ollama and llama.cpp | different template *versions* | compare rendered prompts (`ollama show --template` vs `llama-template`); re-quant or override |
| `enable_thinking` has no effect | stale template in GGUF / always-thinking model | newer quant; or accept it and use `--hidethinking` |
| Garbage first token | template applied twice, or raw text fed to a chat model | single template application; always go through the chat API, not `/v1/completions`, for instruct models |

## Practical rules

1. **Never hand-write role markers.** Let the runtime apply the model's template.
2. **Render before you debug.** One rendered prompt settles 90% of template questions — `llama-template`, `apply_chat_template`, `ollama show --template`.
3. **One system prompt per task**, stored as a file, versioned (it's part of your prompt, not the template — don't conflate them).
4. **Keep quants fresh** — the template is in the GGUF metadata; model families revise templates (reasoning, tools) after release.
5. **Test the stop behavior**: send a trivial prompt and confirm the model ends its turn cleanly (`finish_reason: "stop"`, not `"length"`).

## Sources

- [HF — Chat templating (canonical guide)](https://huggingface.co/docs/transformers/main/en/chat_templating)
- [llama.cpp server README — `--jinja`, `--chat-template*`, `--reasoning*`](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Ollama — Modelfile (`TEMPLATE`, `SYSTEM`)](https://docs.ollama.com/modelfile)
- [vLLM — Reasoning outputs (`--reasoning-parser`, template kwargs)](https://docs.vllm.ai/en/latest/features/reasoning_outputs)
- [This repo — 00-foundations system prompts & chat templates](../../00-foundations/notes/system-prompts-and-chat-templates.md)
- [This repo — 04-llama-cpp/notes/llama-server.md](../../04-llama-cpp/notes/llama-server.md)

## My experiments

- (fill in: e.g. "rendered the same 3-message conversation with `llama-template` vs `transformers.apply_chat_template`: identical except one trailing special token — the GGUF template was one revision behind; after re-pulling the quant, Qwen3.8's `/no_think` toggle worked and thinking stopped leaking into content")
