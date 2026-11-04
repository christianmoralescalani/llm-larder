# Modelfile: building custom models

**Summary.** A Modelfile is Ollama's blueprint format: `FROM` (base model / GGUF / safetensors) + `PARAMETER` (sampling defaults) + `SYSTEM` (persona) + `TEMPLATE` (chat template) + a few extras. `ollama create -f Modelfile` builds it into the blob store. In Docker, the Modelfile (and any GGUF it references) must sit on a mounted volume, because paths resolve **inside the container**.

## Format

```
# comment
INSTRUCTION arguments
```

- Case-insensitive; instructions can be in any order (convention: `FROM` first).
- Multiline values: wrap in `"""`.

## FROM (required)

```dockerfile
FROM llama3.2                    # existing model (name:tag)
FROM qwen3:8b
FROM ./ollama-model.gguf         # GGUF file (relative to the Modelfile, or absolute)
FROM ./model-*.gguf              # split GGUF — wildcard matches all shards
FROM /models/my-model            # safetensors directory (supported architectures)
```

In Docker, a GGUF path like `/models/foo.gguf` requires `-v ~/Downloads/models:/models`.

## PARAMETER

| Parameter | Default | Notes |
|---|---|---|
| `num_ctx` | 2048 | context window (tokens); when unset, the server's `OLLAMA_CONTEXT_LENGTH` (default 4096) applies |
| `temperature` | 0.8 | higher = more creative |
| `top_k` | 40 | limit to the K most likely tokens |
| `top_p` | 0.9 | nucleus sampling threshold |
| `min_p` | 0.0 | minimum probability relative to the top token |
| `repeat_penalty` | 1.0 (off) | >1 penalizes repeats |
| `repeat_last_n` | 64 | look-back window; 0 = off, -1 = whole context |
| `seed` | 0 (random) | fixed seed = reproducible output |
| `stop` | — | stop sequences; repeatable |
| `num_predict` | -1 (unlimited) | max output tokens |
| `draft_num_predict` | 4 (separate draft) | speculative decoding / MTP: max draft tokens per step; 0 = off |

## TEMPLATE

Go `text/template` syntax; variables `{{ .System }}`, `{{ .Prompt }}`, `{{ .Response }}`:

```dockerfile
TEMPLATE """{{ if .System }}<|system|>
{{ .System }}<|end|>{{ end }}{{ if .Prompt }}<|user|>
{{ .Prompt }}<|end|>{{ end }}<|assistant|>
{{ .Response }}"""
```

You rarely need to override it — library models ship with the correct template. Override it when importing a GGUF whose template Ollama can't infer.

## SYSTEM

```dockerfile
SYSTEM """You are a concise code reviewer.
Always cite the line number. Max 3 bullets."""
```

## The rest

```dockerfile
LICENSE """MIT"""                    # license text for the model
MESSAGE user Is Toronto in Canada?   # few-shot history (repeatable)
MESSAGE assistant yes
REQUIRES 0.40.0                      # minimum Ollama version
CAPABILITY decision                  # declare an extra capability (GGUF/MLX)
```

**ADAPTER** (in the old checklist): the LoRA-adapter directive from the Llama-2 era — **no longer in the current Modelfile reference** (verified 2026-11-03). Don't write new Modelfiles with it.

## Docker workflow

1. Put the Modelfile (and the GGUF, if any) on a mounted volume:

```bash
# ~/ollama-work/Modelfile (host) → /work/Modelfile (container)
docker run -d --name ollama --gpus all \
  -v ollama:/root/.ollama \
  -v ~/ollama-work:/work:ro \
  -p 11434:11434 ollama/ollama
```

2. Build:

```bash
docker exec ollama ollama create my-assistant -f /work/Modelfile
```

3. Run it:

```bash
docker exec -it ollama ollama run my-assistant
```

4. Iterate: edit the Modelfile, `create` again (same name = overwrite), or version with `cp`:

```bash
docker exec ollama ollama cp my-assistant my-assistant:v2
```

## Example 1 — persona + parameters from a library model

`Modelfile`:

```dockerfile
FROM qwen3:8b
PARAMETER temperature 0.4
PARAMETER top_p 0.9
PARAMETER num_ctx 8192
SYSTEM """You are a grumpy but helpful DBA.
Answer with the SQL first, then a one-line explanation."""
```

```bash
docker exec ollama ollama create grumpy-dba -f /work/Modelfile
docker exec -it ollama ollama run grumpy-dba "why is my query slow?"
```

## Example 2 — import your own GGUF (no registry)

`Modelfile` (absolute paths, file on the mounted volume):

```dockerfile
FROM /models/Qwen3.8-27B-UD-Q4_K_XL.gguf
PARAMETER temperature 0.6
PARAMETER top_p 0.95
PARAMETER num_ctx 32768
```

```bash
docker exec ollama ollama create qwen38 -f /models/Modelfile
docker exec ollama ollama ls
```

This is how you run any community GGUF (see [01-huggingface](../../01-huggingface/notes/formats-and-quantization.md) for finding them).

## Inspecting

```bash
docker exec ollama ollama show grumpy-dba --modelfile   # the full blueprint
docker exec ollama ollama show grumpy-dba --parameters
docker exec ollama ollama show grumpy-dba --system
```

## Sources

- [Modelfile reference](https://docs.ollama.com/modelfile)
- [Ollama Docker docs](https://docs.ollama.com/docker)

## My experiments

- (fill in: e.g. "imported Qwen3.8-27B GGUF via one-line Modelfile; `show --modelfile` printed the inferred template — matches the HF repo's chat template")
