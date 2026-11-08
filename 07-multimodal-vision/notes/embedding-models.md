# Embedding models — what they are and how to use them (bonus)

**Summary.** An embedding model maps text → a fixed-length vector (768–1,024 dims) so that *similar meaning = nearby vectors*. No generation, no sampling params, no chat template — one forward pass, done. They power **RAG**: embed your documents once, embed the question at query time, take the top-k nearest chunks, stuff them into the LLM's prompt. Locally the workhorses are **nomic-embed-text** (137M, 8k context), **bge-m3** (567M, multilingual + hybrid), and the 2026 class **Qwen3-Embedding** (0.6B–8B, top MTEB scores, Matryoshka = you can shorten the vector).

## How they differ from LLMs

| | LLM (decoder) | Embedding model |
|---|---|---|
| Output | next tokens (generated) | one vector per input |
| Sampling params | yes (temp/top_p/…) | no |
| Chat template | yes | no (plain text in) |
| Speed | slow (decode loop) | fast (single pass, batchable) |
| VRAM | big | small (0.1–8 GB) |

- Embeddings are (near-)symmetric: distance(a, b) ≈ distance(b, a) — that's what makes retrieval work.
- **Cosine similarity** is the default metric; vectors are usually L2-normalized, so cosine = dot product.
- **Prefixes matter**: models like nomic-embed-text want a prefix on the input ("search_query: …" vs "passage: …"). Using the right prefix can roughly double RAG accuracy — see the "Don't Embed Wrong!" video in [videos.md](../videos.md).

## The models

| Model | Params | Dims | Context | Notes |
|---|---|---|---|---|
| nomic-embed-text | 137M | 768 | 8,192 | the 8 GB laptop RAG default; runs on CPU |
| bge-small / bge-m3 | 33M / 567M | 384 / 1024 | 512 / 8,192 | m3 = multilingual + hybrid (sparse + dense) |
| gte-Qwen2-1.5B | 1.5B | 1536 | 32k | strong general |
| Qwen3-Embedding 0.6B / 4B / 8B | — | 1024 (Matryoshka: 32–1024) | 32k | 2026 quality leader; multilingual; instruction-aware |
| multilingual-e5 | 0.3–12B | 384–1024 | 512 | solid multilingual, older |

**Matryoshka** (Qwen3-Embedding): the vector is designed so the first N dims still work — truncate to 256 dims for ~4× less storage with a small quality hit.

## Running them locally

```bash
# Ollama
ollama pull nomic-embed-text
ollama embeddings "what is a KV cache"
curl http://localhost:11434/api/embed -d '{"model":"nomic-embed-text","input":"what is a KV cache"}'

# llama.cpp
llama-embedding -m nomic-embed-text.gguf -p "what is a KV cache"
# or serve it: /v1/embeddings (OpenAI-compatible)

# vLLM
vllm serve Qwen/Qwen3-Embedding-0.6B
curl http://localhost:8000/v1/embeddings -d '{"model":"qwen3-embed","input":"what is a KV cache"}'
```

## RAG in 5 steps

1. **Chunk** your docs (~250–500 tokens, 10–15% overlap).
2. **Embed** every chunk → store vectors + chunk text (sqlite-vec, chroma, or a JSON file for small corpora).
3. **Embed the question** the same way (with the query prefix).
4. **Top-k** nearest (k=3–5) → those chunks are your context.
5. **Prompt the LLM**: `Context:\n"""\n<chunks>\n"""\nQuestion: …` (delimiter pattern from [06 prompting](../06-sampling-and-prompting/notes/prompting-basics.md)).

Rules:

- Embed and query with the **same model** — vectors from different models live in different spaces.
- **Re-embed everything** if you switch models (the old index is worthless).
- Instruction-aware models (Qwen3-Embedding) want the query phrased as an instruction ("Represent this query for retrieval: …").
- Retrieval quality is usually the RAG bottleneck — fix chunking + embedding before blaming the LLM.

## Sources

- [MTEB leaderboard (the benchmark to compare on)](https://huggingface.co/spaces/mteb/leaderboard)
- [nomic-embed-text (HF)](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5)
- [BGE (BAAI, HF)](https://huggingface.co/BAAI)
- [Qwen3-Embedding (HF)](https://huggingface.co/Qwen/Qwen3-Embedding-8B)
- [Ollama — API reference (`/api/embed`)](https://docs.ollama.com/api)
- [This repo — 06 prompting basics (RAG prompt patterns)](../06-sampling-and-prompting/notes/prompting-basics.md)

## My experiments

- (fill in: e.g. "50-question eval over my notes corpus: nomic-embed-text top-5 recall 71%; Qwen3-Embedding-0.6B top-5 recall 84% at 4× the VRAM; adding the `search_query:`/`passage:` prefixes to nomic lifted recall to 78% for free")
