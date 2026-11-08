# References — 08-apis-and-clients

Official docs and the tools this section is based on.

## The OpenAI-compatible API

- [OpenAI API reference (the canonical shape)](https://platform.openai.com/docs/api-reference)
- [OpenAI Python SDK](https://github.com/openai/openai-python)
- [Ollama — OpenAI compatibility](https://docs.ollama.com/api)
- [llama.cpp server README (endpoints, `--api-key`, `--alias`)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [vLLM — OpenAI-compatible server](https://docs.vllm.ai/en/stable/serving/openai_compatible_server/)
- [vLLM — structured output (guided decoding)](https://docs.vllm.ai/en/latest/features/structured_outputs/)

## Frontends

- [Open WebUI — docs / quick start](https://docs.openwebui.com/getting-started/quick-start)
- [Open WebUI — GitHub](https://github.com/open-webui/open-webui)
- [Open WebUI — MCP integrations](https://docs.openwebui.com/guides/users/mcp-integrations)
- [AnythingLLM](https://anythingllm.com)
- [LibreChat — GitHub](https://github.com/danny-avila/LibreChat)
- [SillyTavern — GitHub](https://github.com/SillyTavern/SillyTavern)

## MCP (Model Context Protocol)

- [MCP — spec & docs](https://modelcontextprotocol.io)
- [MCP servers directory](https://modelcontextprotocol.io/servers)
- [MCP Python SDK (FastMCP)](https://github.com/modelcontextprotocol/python-sdk)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Reference servers (filesystem, memory, sqlite, fetch, time)](https://github.com/modelcontextprotocol/servers)

## This repo (cross-references)

- [03-ollama/notes/rest-api.md](../../03-ollama/notes/rest-api.md) — Ollama's native API
- [04-llama-cpp/notes/api.md](../../04-llama-cpp/notes/api.md) — llama.cpp endpoints deep dive
- [05-vllm](../../05-vllm/) — vLLM serving
- [06-sampling-and-prompting/notes/prompting-basics.md](../../06-sampling-and-prompting/notes/prompting-basics.md) — RAG/JSON prompting patterns
