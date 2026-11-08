# Cheatsheet — frontends for local LLMs

**TL;DR.** All four point at your runtime via `base_url` (OpenAI-compatible or Ollama-native). Default: **Open WebUI** (one Docker command, RAG + MCP built in). Deep dive: `notes/frontends.md`.

| You want | Pick | Port |
|---|---|---|
| ChatGPT replacement + RAG + MCP | **Open WebUI** | 3000 |
| Desktop app, RAG zero-config | **AnythingLLM** | 3001 |
| Many backends (local + cloud), teams | **LibreChat** | 3080 |
| Roleplay, characters, lorebooks | **SillyTavern** | 8000 |

## Open WebUI quick start

```bash
# bundled Ollama (GPU)
docker run -d -p 3000:8080 --gpus all \
  -v ollama:/root/.ollama -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY=your-secret-key \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:ollama

# or point at an existing Ollama on the host
docker run -d -p 3000:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

- First load → create admin account.
- Add llama.cpp/vLLM: Admin → Settings → Connections → `http://host.docker.internal:8080/v1` or `:8000/v1`.
- MCP: Admin → Settings → MCP Integrations.
- Back up the `open-webui` volume (chats, users, vector store).

## The one rule

**From a container, the host's runtime is `http://host.docker.internal:PORT`** — on Linux add `--add-host=host.docker.internal:host-gateway`. `localhost` inside the container is the container itself.

## Gotchas

1. Port collisions — check `docker ps` before picking.
2. RAG needs an embedding model reachable from *some* configured backend, or document upload fails obscurely.
3. Slow first message = model load, not the UI (check runtime logs).
4. LAN only — never expose to the internet without auth + TLS (see `10-deployment`).

Verified against Open WebUI 0.7.x, AnythingLLM 1.13.x, LibreChat 0.9.x — 2026-11-08.
