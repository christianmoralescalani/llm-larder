# Videos — 08-apis-and-clients

Curated list. Only videos I've actually checked; timestamps are from the video itself.

## The OpenAI-compatible API

- [Ollama adds OpenAI API support — Learn Data with Mark](https://www.youtube.com/watch?v=Zgb2TjZ5zTk) — short (≈10 min), shows the exact moment a curl to OpenAI gets repointed at `localhost:11434` and the same response shape comes back; then the same swap in the Python `openai` library. Good for the "one client, any backend" mental model. Key moments: [0:56] curl against Ollama, [3:55] replacing the LiteLLM call with `client.chat.completions.create`. My notes: (fill in after watching)

## Frontends

- [Open WebUI Tutorial: A Free ChatGPT for Your Local Ollama Models](https://www.youtube.com/watch?v=yEBT7iYwC5Y) — end-to-end Docker setup of Open WebUI against Ollama. Key moments: [3:52] the docker run command, [4:17] first load, then the feature tour (file upload, RAG/knowledge bases). My notes: (fill in after watching)

## MCP

- [MCP in 10 Minutes (Model Context Protocol + Free Lab) — KodeKloud](https://www.youtube.com/watch?v=dyt-bhxrrbk) — the clearest short explainer I found: the N×N problem first, then architecture, then building a real server. Key moments: [00:17] the N×N problem, [01:09] how it works, [04:19] server architecture, [07:42] building the flight-booking server, [10:28] testing with a client. My notes: (fill in after watching)
- [The Model Context Protocol (MCP)](https://www.youtube.com/watch?v=CQywdSdi5iA) — longer podcast-style deep dive with the people who built it; good for the "why" and the ecosystem direction. Key moments: [0:35] core components (tools/resources/prompts), [3:13] history, [13:23] tips for getting started, [14:57] favorite MCP servers. My notes: (fill in after watching)
