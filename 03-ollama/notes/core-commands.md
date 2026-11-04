# Core commands (via `docker exec`)

**Summary.** The server runs inside the container; the CLI is just a client. From the host, every command is `docker exec ollama ollama <cmd>` (add `-it` only for interactive `run`). The day-to-day set: `run`, `pull`, `ls`, `ps`, `stop`, `show`, `create`, `serve`, `help` — plus `launch` (new in 2026) for wiring Ollama into coding agents.

## The wrapper

```bash
# non-interactive (no TTY needed)
docker exec ollama ollama ls

# interactive (needs a TTY)
docker exec -it ollama ollama run llama3.2
```

## Command table (verified 2026-11-03)

| Command | What it does |
|---|---|
| `ollama run <model> [prompt]` | chat (interactive, or one-shot with a prompt); image input; `/set` in session |
| `ollama pull <model>[:tag]` | download a model from the registry |
| `ollama push <model>[:tag]` | upload to the registry (needs `ollama signin`) |
| `ollama ls` (`list`) | list local models |
| `ollama ps` | list *loaded* models (VRAM, GPU/CPU split, expiry) |
| `ollama stop <model>` | unload a model from memory |
| `ollama rm <model>` | delete a local model |
| `ollama create -f Modelfile` | build a custom model (see [modelfile.md](modelfile.md)) |
| `ollama show [flags] <model>` | inspect: `--modelfile --system --template --parameters --license` |
| `ollama cp <src> <dst>` | copy/rename/tag a local model (instant) |
| `ollama serve` | start the server (already PID 1 in the container) |
| `ollama launch [integration]` | configure + launch an app (Claude Code, Codex, VS Code, …) |
| `ollama signin` / `signout` | registry auth (ollama.com) |
| `ollama help` / `ollama -v` | help / version |

Note on `bench`: the original plan mentioned `ollama bench` — **it does not exist in the current CLI** (verified against the [CLI reference](https://docs.ollama.com/cli)). To measure speed, use the API timing fields (see [rest-api.md](rest-api.md)) or `llama-bench` ([04-llama-cpp](../../04-llama-cpp/)).

## `run` — chat

```bash
# interactive REPL
docker exec -it ollama ollama run llama3.2

# one-shot
docker exec ollama ollama run llama3.2 "Explain KV cache in one sentence."

# with an image (path is INSIDE the container — mount the dir first)
docker exec -it ollama ollama run gemma3 /models/img.png "what's in this image?"

# piped stdin
echo "Summarize: <text>" | docker exec ollama ollama run llama3.2
```

In-REPL commands: `/set parameter num_ctx 8192`, `/set parameter temperature 0.5`, `/show`, `/save [name]` (saves session changes as a new model), `/clear`, `/bye`. Multiline input: wrap in `"""`.

## `pull` / `ls`

```bash
docker exec ollama ollama pull llama3.2        # default tag :latest
docker exec ollama ollama pull qwen3:30b-a3b   # explicit tag
docker exec ollama ollama ls
# NAME             ID              SIZE      MODIFIED
# llama3.2:latest  501a7d391971    2.0 GB    2 minutes ago
```

## `ps` / `stop`

```bash
docker exec ollama ollama ps
# NAME            ID              SIZE     PROCESSOR    UNTIL
# llama3.2:latest 501a7d391971    1.7 GB   100% GPU     5 minutes from now

docker exec ollama ollama stop llama3.2
```

`PROCESSOR`: `100% GPU` = fully on GPU; `100% CPU` = RAM only (slow); `48%/52% CPU/GPU` = split (partial offload).

## `show`

```bash
docker exec ollama ollama show llama3.2 --modelfile   # the full blueprint
docker exec ollama ollama show llama3.2 --parameters  # sampling defaults
docker exec ollama ollama show llama3.2 --system      # system prompt
docker exec ollama ollama show llama3.2 --template    # chat template
docker exec ollama ollama show llama3.2 --license
```

## `serve`

Inside the container, `ollama serve` is already the main process. You only pass it explicitly when starting the container with a custom command:

```bash
docker run -d --name ollama --gpus all -v ollama:/root/.ollama \
  -p 11434:11434 ollama/ollama serve
```

`ollama serve --help` prints the full list of environment variables — the authoritative config reference (see [configuration.md](configuration.md)).

## `launch` (new in 2026)

```bash
docker exec -it ollama ollama launch                    # interactive picker
docker exec -it ollama ollama launch claude             # e.g. Claude Code
docker exec -it ollama ollama launch codex --model qwen3
docker exec ollama ollama launch droid --config         # configure only
```

Wires Ollama into external apps (Claude Code, Codex, Copilot CLI, VS Code, Droid, …).

## `help` / version

```bash
docker exec ollama ollama help
docker exec ollama ollama -v                    # client version
curl http://localhost:11434/api/version         # server version (from the host)
```

## Sources

- [Ollama CLI reference](https://docs.ollama.com/cli)
- [Ollama Docker docs](https://docs.ollama.com/docker)

## My experiments

- (fill in: e.g. "`ollama ps` after first run: 100% GPU, UNTIL 5 minutes from now — matches OLLAMA_KEEP_ALIVE default")
