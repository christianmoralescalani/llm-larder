# Videos — 07-multimodal-vision

Curated video list for this topic. Format:

- [Title](url) — why it is good, key timestamps, my notes

## Videos

### Vision models

- [Video Understanding with Qwen2.5-VL-7B-Instruct — Run Locally](https://www.youtube.com/watch?v=blCXng9yYBw) — install + run Qwen2.5-VL locally, then point it at an MP4 (video run at ~7:09). Good "first local video model" walkthrough.
- [Ollama Local AI Vision: Qwen 2.5 VL and Gemma 3 QAT Test](https://www.youtube.com/watch?v=1kchCV0lWBU) — side-by-side Qwen2.5-VL vs Gemma 3 in Ollama; on-screen token stats (~9:57: 511 total tokens, 33 t/s) make the image-token cost concrete.
- [Top Vision Models 2025: Qwen 2.5 VL, Moondream & SmolVLM (Fine-Tuning & Benchmarks)](https://www.youtube.com/watch?v=f-nF-48rDIM) — architecture explained (encoder → adapter → LLM at ~14:41), dynamic token counts (~51:01), plus a live fine-tune of Qwen2.5-VL. Long but the best single "how + what" video.

### Audio

- [100% Local AI Speech to Speech with RAG — Low Latency (Mistral 7B, Faster Whisper++)](https://www.youtube.com/watch?v=VpB6bxh4deM) — the full local pipeline: Whisper ASR → LLM → TTS, all offline, with latency numbers. The "speech-to-speech at home" watch.

### Embeddings & RAG

- [RAG Series — How to Generate Embeddings with Ollama (Ep. 4)](https://www.youtube.com/watch?v=IM6bq3BUQAI) — nomic-embed-text via Ollama in Python, chunking → embedding → vector store, step by step.
- [Use Ollama Embeddings with PostgreSQL (Tutorial)](https://www.youtube.com/watch?v=3GPLb12FfNw) — pgvector + Ollama embeddings, RAG as a SQL query (similarity search at ~12:04). Good if you want the vector DB to be Postgres.
- [Crazy FAST RAG — Ollama, Nomic Embedding, Groq](https://www.youtube.com/watch?v=TMaQt8rN5bE) — fast local embedding + RAG with a chat UI; pragmatic, not academic.
- [Don't Embed Wrong! (Nomic prefixes)](https://www.youtube.com/watch?v=76EIC_RaDNw) — short and important: input prefixes (`search_query:` / `passage:`) can ~2× RAG accuracy. Watch before building your first RAG.

## Playlists

- (fill in: e.g. Hugging Face "Smol AI" course — VLM episodes; llama.cpp official channel)
