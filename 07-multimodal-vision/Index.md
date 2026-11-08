# Index — 07-multimodal-vision

**How to read this folder.** Vision is the main topic; embeddings and audio are the bonus items. Read in order: understand how it works → know the models → run one → know the VRAM cost. The two bonus notes are self-contained — jump straight to them if you need RAG (embeddings) or speech (audio).

## Start here

`notes/how-vision-llms-work.md` — image → patches → visual tokens, and why an image "costs" tokens.

## Reading order

1. `notes/how-vision-llms-work.md` — the pipeline, visual tokens, context impact
2. `notes/vision-models.md` — the landscape: Qwen3-VL, Gemma 3, Llama 3.2, MiniCPM-V — who's who
3. `notes/running-vision-models.md` — actually running them in Ollama / llama.cpp / vLLM, with working examples
4. `notes/vision-vram.md` — why vision needs more headroom (encoder, visual KV, activations)
5. `notes/embedding-models.md` — *(bonus)* embeddings: nomic-embed, bge, Qwen3-Embedding, RAG in 5 steps
6. `notes/audio-models.md` — *(bonus)* Whisper / faster-whisper / whisper.cpp, Qwen2-Audio

## Quick reference (cheatsheets)

- `cheatsheets/image-token-cost.md` — how many tokens an image costs, by resolution/model
- `cheatsheets/vision-models.md` — the model table: sizes, context, what they're good at
- `cheatsheets/audio-embeddings.md` — the bonus commands: embeddings + audio pipelines

## Sources & videos

- `docs/references.md` — model reports, runtime docs
- `videos.md` — curated videos
