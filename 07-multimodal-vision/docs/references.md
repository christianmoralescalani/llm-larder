# References — 07-multimodal-vision

Official docs and the tools this section is based on.

## Vision models

- [HF — Vision language models (LLaVA architecture, projector)](https://huggingface.co/blog/vlms)
- [Qwen2-VL blog (dynamic resolution, min/max pixels)](https://qwenlm.github.io/blog/qwen2-vl)
- [Qwen2.5-VL blog (video, grounding)](https://qwenlm.github.io/blog/qwen2.5-vl)
- [Qwen3-VL (OpenLM)](https://openlm.ai/qwen3-vl)
- [Gemma 3 technical report (SigLIP, 256 tokens, Pan & Scan)](https://arxiv.org/html/2503.19786v1)
- [Google — Gemma vision (token budgets 70–1,120)](https://ai.google.dev/gemma/docs/capabilities/vision)
- [Meta — Llama 3.2 (vision models, 128k context)](https://ai.meta.com/blog/llama-3-2-connect-2024-vision-edge-mobile-devices)
- [Meta — Llama 4 (natively multimodal)](https://ai.meta.com/blog/meta-llama-4-multimodal-intelligence)
- [MiniCPM-V (OpenBMB)](https://github.com/OpenBMB/MiniCPM-V)
- [SmolVLM (Hugging Face)](https://huggingface.co/blog/smolvlm)
- [Moondream](https://github.com/vikhyat/moondream)
- [Pixtral (Mistral)](https://mistral.ai/news/pixtral-large)
- [InternVL](https://github.com/OpenGVLab/InternVL)
- [Efficient LLaMA-3.2-Vision (measured visual token counts)](https://arxiv.org/html/2504.00557v1)

## Runtimes (multimodal)

- [llama.cpp — multimodal (mtmd) README](https://github.com/ggml-org/llama.cpp/blob/master/tools/mtmd/README.md)
- [llama.cpp server README (`--mmproj*`, `--image`, video)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Ollama — vision models (docs)](https://github.com/ollama/ollama/blob/main/docs/vision.md)
- [vLLM — multimodal inputs](https://docs.vllm.ai/en/stable/features/multimodal_inputs/)
- [vLLM — supported models (vision)](https://docs.vllm.ai/en/stable/models/supported_models/)

## Audio

- [Whisper (OpenAI, GitHub)](https://github.com/openai/whisper)
- [faster-whisper (CTranslate2)](https://github.com/SYSTRAN/faster-whisper)
- [whisper.cpp](https://github.com/ggml-org/whisper.cpp)
- [Qwen2-Audio](https://github.com/QwenLM/Qwen2-Audio)

## Embeddings

- [MTEB leaderboard](https://huggingface.co/spaces/mteb/leaderboard)
- [nomic-embed-text (HF)](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5)
- [BGE (BAAI, HF)](https://huggingface.co/BAAI)
- [Qwen3-Embedding (HF)](https://huggingface.co/Qwen/Qwen3-Embedding-8B)
- [Ollama — API reference (`/api/embed`)](https://docs.ollama.com/api)

## In-repo

- `01-huggingface/notes/mmproj-vision.md` — the mmproj file (what's inside, naming, precision)
- `02-hardware-and-vram/cheatsheets/vram-budget.md` — the base VRAM math this section extends
- `03-ollama/notes/vision-models.md` — Ollama vision (CLI + API)
- `04-llama-cpp/notes/llama-server.md` — server flags (incl. `--mmproj*`)
- `06-sampling-and-prompting/notes/prompting-basics.md` — RAG prompt patterns used with embeddings
