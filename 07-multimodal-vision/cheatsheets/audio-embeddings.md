# Cheatsheet — audio & embedding commands (bonus)

**TL;DR.** Audio: **Whisper** (ASR, speech → text, not an LLM) — faster-whisper or whisper.cpp; **Qwen2-Audio** for audio *understanding*. Embeddings: text → vector for RAG — nomic-embed-text (laptop), bge-m3 (multilingual), Qwen3-Embedding (2026 quality leader). No sampling params on either. Deep dives: `notes/audio-models.md`, `notes/embedding-models.md`.

## Whisper

```bash
# faster-whisper (best local default)
pip install faster-whisper
python3 -c "
from faster_whisper import WhisperModel
m = WhisperModel('large-v3', device='cuda', compute_type='float16')
segs, info = m.transcribe('audio.mp3', vad_filter=True)
print(''.join(s.text for s in segs))"

# whisper.cpp (no Python)
./whisper-cli -m ggml-large-v3.bin -f audio.wav -l en
```

Sizes: tiny 39M · base 74M · small 244M · medium 769M · **large-v3 1.55B (~3 GB VRAM)**. VAD filter = skip silence (big speedup). Force the language (`-l en`) for accuracy.

## Embeddings

```bash
# Ollama
ollama pull nomic-embed-text
ollama embeddings "what is a KV cache"
curl http://localhost:11434/api/embed -d '{"model":"nomic-embed-text","input":"..."}'

# llama.cpp
llama-embedding -m nomic-embed-text.gguf -p "what is a KV cache"

# vLLM (OpenAI-compatible /v1/embeddings)
vllm serve Qwen/Qwen3-Embedding-0.6B
```

| Model | Params | Dims | Context | Pick when |
|---|---|---|---|---|
| nomic-embed-text | 137M | 768 | 8k | 8 GB laptop RAG |
| bge-m3 | 567M | 1024 | 8k | multilingual / hybrid |
| Qwen3-Embedding 0.6B/4B/8B | — | 1024 (Matryoshka) | 32k | best quality, 2026 |

## RAG in 5 steps

chunk (250–500 tok, ~10% overlap) → embed all chunks → embed query (with prefix) → top-k (3–5) → stuff into prompt with `"""` delimiters. Same model for docs and queries; re-embed everything if you switch models; use the model's input prefixes (`search_query:` / `passage:`).

Verified against Ollama/llama.cpp/vLLM docs — 2026-11-08.
