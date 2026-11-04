# Cheatsheet — Ollama commands

**TL;DR.** In Docker, everything is `docker exec ollama ollama <cmd>` (add `-it` only for interactive `run`). Day-to-day: `run` (chat), `pull` (download), `ls` (list), `ps` (loaded + VRAM), `stop` (unload), `rm` (delete), `create -f` (build), `show` (inspect). No `bench` in the current CLI — measure tok/s from the API timing fields. Verified on `ollama/ollama:0.40.14`, 2026-11-03.

## Command table

| Command | What it does |
|---|---|
| `run <model> [prompt] [img.png]` | chat; one-shot with a prompt; image input; `/set parameter …` in session; `/save` to fork |
| `pull <model>[:tag]` | download from the registry |
| `push <model>[:tag]` | upload (needs `signin`) |
| `ls` / `list` | local models (name, id, size) |
| `ps` | loaded models: size, PROCESSOR (GPU/CPU split), UNTIL |
| `stop <model>` | unload from memory |
| `rm <model>` | delete (all tags if bare name) |
| `create -f Modelfile` | build a custom model |
| `cp <src> <dst>` | rename/tag (instant, no blob copy) |
| `show [--modelfile\|--system\|--template\|--parameters\|--license] <model>` | inspect |
| `serve` | start the server (already PID 1 in the container) |
| `launch [app] [--model m]` | wire Ollama into Claude Code / Codex / VS Code / … |
| `signin` / `signout` | registry auth |
| `help` / `-v` | help / version |

## Docker wrapper

```bash
docker exec ollama ollama ls                        # non-interactive
docker exec -it ollama ollama run llama3.2          # interactive (TTY)
echo "hi" | docker exec ollama ollama run llama3.2  # piped one-shot
```

## In-REPL (`ollama run`)

```
/set parameter num_ctx 8192
/set parameter temperature 0.5
/show            # current model + parameters
/save my-model   # save the session's changes as a new model
/clear           # clear history
/bye             # exit
```

Multiline input: wrap in `"""`.

## Measuring speed (no `ollama bench`)

```bash
curl -s http://localhost:11434/api/generate -d '{"model":"llama3.2","prompt":"Tell me a story.","stream":false}' \
  | jq '{decode_tps: (.eval_count / (.eval_duration/1e9)), prefill_tps: (.prompt_eval_count / (.prompt_eval_duration/1e9))}'
```

For real benchmarks: `llama-bench` ([04-llama-cpp](../../04-llama-cpp/)).

## Verified on

- `ollama/ollama:0.40.14` (Docker Hub `latest`), 2026-11-03, against the [CLI reference](https://docs.ollama.com/cli).
