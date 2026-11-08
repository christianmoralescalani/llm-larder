# MCP servers — giving your local LLM tools (bonus)

**Summary.** MCP (Model Context Protocol) is an open standard — JSON-RPC 2.0 over stdio or HTTP — that lets an LLM **application** (the "client": Open WebUI, LM Studio, Claude Desktop, your own script) talk to **servers** that expose three things: **tools** (functions the model can call), **resources** (read-only data, e.g. files/DB rows), and **prompts** (saved templates). The point: one server, many clients; one client, many servers — the "USB-C for LLMs". For local use it means your local model can read your files, query your database, and hit local APIs without you hand-rolling tool-call parsing.

## The architecture

```text
┌─────────────────────────────┐
│  MCP client (the app)       │
│  Open WebUI / LM Studio /   │
│  Claude Desktop / your code │
└──────────────┬──────────────┘
               │ JSON-RPC 2.0 (stdio subprocess, or HTTP)
   ┌───────────┼───────────────┬────────────────┐
   ▼           ▼               ▼                ▼
┌───────┐  ┌────────┐    ┌──────────┐     ┌──────────┐
│ tools │  │resources│   │ prompts  │     │ another  │
│server │  │ server  │    │ server   │     │ server…  │
└───────┘  └────────┘    └──────────┘     └──────────┘
```

- **The client owns the loop**: it shows the model's tool calls, executes them via the server, feeds results back. The *model* just emits a structured "call this tool with these args" — the client does the rest.
- **Transports**: `stdio` (client spawns the server as a subprocess — the local default) and **streamable HTTP** (server runs as a service, multiple clients).
- **Capability negotiation**: at connect, client and server advertise what they support (tools/resources/prompts/sampling); a client that supports none of them just won't see them.
- **The model doesn't talk to the server directly** — it never sees the protocol. That's why any model (local or cloud) works with any MCP server.

## What a tool looks like (the model's view)

```json
{
  "name": "get_weather",
  "description": "Get current weather for a city",
  "inputSchema": {
    "type": "object",
    "properties": {"city": {"type": "string"}},
    "required": ["city"]
  }
}
```

That's the same shape as OpenAI `tools` (see [curl cookbook — tool calling](curl-cookbook.md)) — MCP generalizes it and adds discovery (`tools/list`), resources (`resources/read`), and prompts (`prompts/get`).

## Running local MCP servers

### With a GUI client

**Open WebUI**: Admin → Settings → MCP Integrations → add a server (command + args for stdio, or URL for HTTP). Its tools appear as usable functions in chat.

**LM Studio**: Settings → MCP Servers → add command/args; tools show up in the chat tool picker.

**Claude Desktop** (the reference client): edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/me/notes"]
    }
  }
}
```

Restart the app → the tool icon appears in the chat input.

### Good first servers (all local, no API keys)

| Server | What it does | Run with |
|---|---|---|
| `server-filesystem` | read/write/list files in allowed dirs | `npx -y @modelcontextprotocol/server-filesystem /path` |
| `server-memory` | knowledge graph the model can query | `npx -y @modelcontextprotocol/server-memory` |
| `server-sqlite` | query a SQLite DB | `uvx mcp-server-sqlite --db-path notes.db` |
| `server-fetch` | fetch + extract web pages (needs network) | `uvx mcp-server-fetch` |
| `server-time` | current time / timezone math | `uvx mcp-server-time` |

`uvx` = run a Python package as a one-off (from [uv](https://github.com/astral-sh/uv)); `npx` = the Node equivalent. Browse more at [modelcontextprotocol.io/servers](https://modelcontextprotocol.io/servers).

### Writing your own (Python, ~20 lines)

```python
# pip install "mcp[cli]"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools")

@mcp.tool()
def gpu_stats() -> str:
    """Return current GPU memory usage."""
    import subprocess
    out = subprocess.run(
        ["nvidia-smi", "--query-gpu=memory.used,memory.total", "--format=csv,noheader"],
        capture_output=True, text=True,
    )
    return out.stdout.strip()

if __name__ == "__main__":
    mcp.run()  # stdio transport
```

Wire it into any client as `{"command": "python", "args": ["mymcp.py"]}`. The docstring becomes the tool description the model sees — write it like an API doc.

## MCP vs native tool calling

| | Native `tools` (OpenAI API) | MCP |
|---|---|---|
| Who defines tools | you, per request | servers, reusable across clients |
| Who executes | **your code** (the client) | the **server** |
| Discovery | none — you list them | `tools/list` at connect |
| Best for | your own app's functions | shared/local utilities, GUI clients |

They compose: a client can expose MCP tools *through* the native `tools` field to the model. For a local setup, MCP is mostly about **GUI clients** (Open WebUI/LM Studio) — in your own Python code, native `tools` + a few functions is simpler.

## Gotchas

1. **Small models + many tools = confusion** — every tool's name+description+schema eats context and attention. 3–5 focused tools beat 20; small models pick the wrong tool or hallucinate args.
2. **The docstring is the prompt** — vague tool descriptions → wrong calls. Test with the actual model you'll use.
3. **stdio servers die silently** — if the subprocess crashes, the client may just show "tool unavailable"; check the client's logs.
4. **Permissions are the server's problem** — `server-filesystem` only touches the dirs you pass it; a malicious/buggy server can do whatever it's allowed to. Run local servers with least privilege.
5. **Not all local clients support MCP yet** — check your frontend's release notes; llama.cpp/vLLM/Ollama *servers* don't speak MCP (they're the model side, not the client side).
6. **HTTP servers need auth if on LAN** — stdio is inherently local; a streamable-HTTP server on 0.0.0.0 is an exposed API.

## Sources

- [Model Context Protocol — spec & docs](https://modelcontextprotocol.io)
- [MCP servers directory](https://modelcontextprotocol.io/servers)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Open WebUI — MCP integrations](https://docs.openwebui.com/guides/users/mcp-integrations)
- [This repo — 08 curl cookbook (native tool calling)](curl-cookbook.md)

## My experiments

- (fill in: e.g. "Open WebUI + Qwen3.8-27B + filesystem MCP server over my notes dir: 'find my vLLM benchmark numbers' → correct file + section in one tool call; with 8 servers enabled the same question failed twice (wrong tool) — dropped to 3 servers and it was reliable; my 20-line nvidia-smi server worked in LM Studio and Open WebUI unchanged")

