# Model management: pull, create, tags, registries

**Summary.** Ollama models are content-addressed blobs (sha256) under `/root/.ollama/models` in the container. `pull` fetches from a registry (ollama.com by default), `create` builds a new model from a Modelfile or raw weights, `cp` renames/tags, `rm` deletes, `push` uploads. Tags (`name:tag`) are just labels on the same blob store — cheap and instant.

## The model store

```bash
docker exec ollama ollama ls
docker exec ollama sh -c 'ls -la /root/.ollama/models'   # manifests + blobs
```

- Each model = a manifest (JSON) + content-addressed blobs under `blobs/sha256-…`.
- Shared layers are deduplicated: two tags pointing at the same model don't double disk usage.
- Disk check: `df -h` on the volume's backing disk (or `docker system df`).

## pull

```bash
docker exec ollama ollama pull llama3.2                 # :latest
docker exec ollama ollama pull qwen3:30b-a3b            # explicit tag
docker exec ollama ollama pull user/my-model:v1         # someone else's model
docker exec ollama ollama pull my-registry.local:5000/my-model   # custom registry
```

Browse the registry: [ollama.com/library](https://ollama.com/library). Pulls are resumable; a killed container mid-pull → re-run `pull` to finish.

## create

Three sources (details in [modelfile.md](modelfile.md)):

```bash
# 1. from an existing model + tweaks (Modelfile with FROM <model>)
docker exec ollama ollama create my-assistant -f /work/Modelfile

# 2. from a GGUF file (Modelfile with FROM /path/model.gguf)
docker exec ollama ollama create my-qwen -f /models/Modelfile-qwen

# 3. from a safetensors directory (Modelfile with FROM /path/to/dir)
docker exec ollama ollama create my-model -f /work/Modelfile-st
```

The Modelfile (and any GGUF/safetensors it references) must be readable **inside the container** — mount the directory.

## Tags and copies

```bash
docker exec ollama ollama cp llama3.2 llama3.2:fast         # new tag, same model
docker exec ollama ollama cp llama3.2:fast llama3.2:fast-v2
```

`cp` is instant (no blob copy) — it just adds a manifest. To make a real variant, `create` from it.

## delete

```bash
docker exec ollama ollama rm llama3.2:fast
docker exec ollama ollama rm llama3.2          # all tags of the name
```

Orphaned blobs are pruned automatically (disable with `OLLAMA_NOPRUNE=1` if you're mid-experiment).

## push (registry)

```bash
docker exec -it ollama ollama signin            # ollama.com account (ed25519 keypair)
docker exec ollama ollama push christian/my-model:v1
docker exec ollama ollama signout
```

- The keypair lives at `/root/.ollama/id_ed25519(.pub)` — with a mounted volume it persists across container recreations.
- Pushing to a **private registry**: use a fully-qualified name (`my-registry.local:5000/user/model`), or set `OLLAMA_REGISTRY` as the default (see [configuration.md](configuration.md)).

## Custom registries

```bash
# default registry for pull/push:
docker run … -e OLLAMA_REGISTRY=my-registry.local:5000 ollama/ollama
# or per-pull with a fully-qualified name:
docker exec ollama ollama pull my-registry.local:5000/user/model
```

## Choosing what to pull (quick guide)

- Chat: `llama3.2` (3B, fast), `qwen3:8b`, `gemma3:12b`
- Coding: `qwen3:30b-a3b` (MoE — big quality, small active params)
- Vision: `llava`, `gemma3` (see [vision-models.md](vision-models.md))
- Embeddings: `nomic-embed-text`
- Size sanity check before pulling: [02-hardware-and-vram cheatsheet](../../02-hardware-and-vram/cheatsheets/vram-budget.md)

## Sources

- [Ollama CLI reference](https://docs.ollama.com/cli)
- [Ollama library](https://ollama.com/library)
- [Modelfile reference](https://docs.ollama.com/modelfile)

## My experiments

- (fill in: e.g. "`cp` a 27B model to a new tag: instant, `ls` shows both, disk unchanged")
