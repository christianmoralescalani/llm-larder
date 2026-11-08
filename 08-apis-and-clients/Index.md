# Index — 08-apis-and-clients

**How to read this folder.** This is the *client* layer: one API shape that works against Ollama, llama.cpp, and vLLM. Read in the order you'd actually use it: understand the API → talk to it with curl → talk to it with Python → put a GUI in front → give it tools. The cheatsheets mirror the three main notes.

## Start here

`notes/openai-compatible-api.md` — why this shape won, what works on each runtime, and the gotchas that bite when you switch.

## Reading order

1. `notes/openai-compatible-api.md` — the common ground: endpoints, support matrix, gotchas
2. `notes/curl-cookbook.md` — health, chat, SSE streaming, JSON mode, tools, images — no SDK
3. `notes/python-clients.md` — the `openai` SDK: sync, async, multi-runtime config, reusable glue
4. `notes/frontends.md` — Open WebUI, AnythingLLM, LibreChat, SillyTavern: install + point at your runtime
5. `notes/mcp-servers.md` — *(bonus)* MCP: giving your local LLM tools, writing your own server

## Quick reference (cheatsheets)

- `cheatsheets/openai-api.md` — base_url/model per runtime + endpoint & feature matrices
- `cheatsheets/curl-cookbook.md` — every recipe, copy-paste ready
- `cheatsheets/frontends.md` — the four frontends: install commands, ports, choosing table

## Sources & videos

- `docs/references.md` — API references, frontend repos, MCP spec
- `videos.md` — curated videos
