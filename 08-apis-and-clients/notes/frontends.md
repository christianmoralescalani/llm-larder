# Frontends — GUIs for local LLMs

**Summary.** A frontend is a chat UI that speaks the OpenAI-compatible (or Ollama-native) API: **Open WebUI** is the default pick (ChatGPT-like, RAG/pipelines/MCP built in, Docker one-liner), **AnythingLLM** is the desktop app with built-in RAG/vector DB, **LibreChat** is the multi-backend power user's choice, **SillyTavern** is the roleplay/storytelling specialist. All of them can point at Ollama, llama.cpp, and vLLM — the difference is what they do *around* the chat. One-page version: `cheatsheets/frontends.md`.

## Open WebUI (the default pick)

ChatGPT-style UI, accounts, file upload + RAG (built-in embedding + vector store), "pipelines" (custom functions), MCP support, and it can proxy to **any** OpenAI-compatible backend.

```bash
# single container with Ollama bundled (GPU)
docker run -d -p 3000:8080 --gpus all \
  -v ollama:/root/.ollama -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY=your-secret-key \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:ollama

# or point at an EXISTING Ollama on the host
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

- Web UI at http://localhost:3000 — create the admin account on first load.
- **Adding backends**: Admin → Settings → Connections → add `http://host.docker.internal:8080/v1` (llama.cpp) or `:8000/v1` (vLLM) as OpenAI connections; models show up in the picker.
- **Data**: the `open-webui` volume holds chats, users, and the vector store — back it up.
- `:cuda` image puts its own local models (embeddings, reranker, Whisper STT) on the GPU.

## AnythingLLM (desktop + Docker)

Desktop app (Win/mac/Linux) or Docker; **built-in** vector DB + embedding model, so RAG works with zero external services. Good "one app does everything" pick.

- Install: download from [anythingllm.com](https://anythingllm.com) or `docker run -d -p 3001:3001 -v ~/anythingllm:/app/server/storage mbinsei/anythingllm`.
- **Backend config**: Settings → LLM Provider → Ollama / OpenAI-compatible (base_url + model). Same for the embedding provider.
- Workspaces = folders of documents + chat history with RAG over them.
- Limits vs Open WebUI: weaker multi-user story, fewer power features (no MCP out of the box in older versions — check current release).

## LibreChat (multi-backend power user)

Node.js app that manages **many** backends (local + cloud) with per-chat model switching, RAG, and a clean UI.

```bash
# Docker (compose is the recommended path — see repo)
docker run -d -p 3080:3080 \
  -v librechat:/app/.data \
  -e LOCAL_API_KEY=local-key -e LOCAL_MODEL=qwen3.8 \
  -e LOCAL_MODEL_BASE_URL=http://host.docker.internal:8080/v1 \
  ghcr.io/danny-avila/librechat:latest
```

- `.env`-driven: each backend is a block (`LOCAL_*`, `OLLAMA_*`, `OPENAI_*`…) — you can mix Ollama + llama.cpp + vLLM + cloud in one install.
- Heavier to configure than Open WebUI; the payoff is backend flexibility + auth for teams.

## SillyTavern (roleplay & storytelling)

The specialist: character cards, world info/lorebooks, multi-character chats, image generation hooks, extensive theming. Not a "productivity" UI.

- Install: download the zip (or Docker), run `start.sh`/`start.bat`, open http://localhost:8000.
- **Backend config**: Settings → Connections → add Ollama / OpenAI-compatible endpoint + model; per-character model overrides.
- Pairs well with any runtime; the character/world-info layer is what you're paying for.

## Choosing

| You want | Pick |
|---|---|
| ChatGPT replacement + RAG + MCP, one Docker command | **Open WebUI** |
| Desktop app, RAG with zero config | **AnythingLLM** |
| Many backends (local + cloud), team auth | **LibreChat** |
| Roleplay, characters, lorebooks | **SillyTavern** |

All four are free/open-source and all four point at your local runtime via `base_url` — try Open WebUI first, switch only if its gaps hurt.

## Frontend gotchas

1. **`host.docker.internal`** — from a frontend container, the host's Ollama/llama.cpp is at `http://host.docker.internal:PORT` (Linux needs `--add-host=host.docker.internal:host-gateway`); `localhost` inside the container is the container itself.
2. **Port collisions** — Open WebUI 3000, AnythingLLM 3001, LibreChat 3080, SillyTavern 8000; check `docker ps` before picking.
3. **Embedding model mismatch** — the frontend's RAG needs an embedding model available on *some* backend you've configured; if it's missing, document upload fails obscurely.
4. **Model name display** — frontends show whatever `GET /v1/models` returns; llama.cpp without `--alias` shows the GGUF filename (ugly but functional).
5. **First-load delay** — the frontend is fast; the *model* isn't. A slow first message is usually model load, not the UI (check the runtime's logs).
6. **Don't expose to the internet** — these UIs are for LAN/local; if you must, put auth + TLS in front (see [10-deployment](../10-deployment/)).

## Sources

- [Open WebUI — docs / quick start](https://docs.openwebui.com/getting-started/quick-start)
- [Open WebUI (GitHub)](https://github.com/open-webui/open-webui)
- [AnythingLLM](https://anythingllm.com)
- [LibreChat (GitHub)](https://github.com/danny-avila/LibreChat)
- [SillyTavern (GitHub)](https://github.com/SillyTavern/SillyTavern)

## My experiments

- (fill in: e.g. "Open WebUI `:ollama` image + Qwen3.8-27B on :8080 via OpenAI connection: chat, RAG over my notes (nomic-embed), and MCP filesystem server all worked; first message after idle took 14 s (model load), ~0.8 s after warmup; AnythingLLM's built-in RAG was the fastest to demo but its model picker was fiddlier with two local backends")
