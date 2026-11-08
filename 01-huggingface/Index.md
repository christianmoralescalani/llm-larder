# Index — 01-huggingface

**How to read this folder.** This is the *shopping* folder: finding, evaluating, and downloading models. Read the notes in order (search → understand formats → inspect → tools); the two "only if" notes at the end are bridges to other topics. Cheatsheets are the ones you'll re-open every time you download something.

## Start here

`notes/searching-models.md` — how to actually find a model worth running (search terms, filters, reading a model card).

## Reading order

1. `notes/searching-models.md` — finding models: search, filters, model cards
2. `notes/formats-and-quantization.md` — what the Hub listings mean (GGUF vs safetensors, quant suffixes) — builds on [00-foundations](../00-foundations/notes/model-formats-and-quantization.md)
3. `notes/inspecting-weights.md` — checking a model *before* you commit to a 15 GB download
4. `notes/useful-tools.md` — the extra tools for Hub work (tokenizers, file browsers, size math)
5. `notes/mmproj-vision.md` — the vision projector file *(only if you're doing vision — pairs with [07-multimodal-vision](../07-multimodal-vision/))*
6. `notes/mtp-models.md` — MTP models on the Hub *(only if you run MTP/speculative decoding — pairs with [04-llama-cpp](../04-llama-cpp/notes/speculative-decoding.md))*

## Quick reference (cheatsheets)

- `cheatsheets/hf-search.md` — search patterns that work
- `cheatsheets/hf-download.md` — `hf` CLI downloads (single file, resume, specific quant)
- `cheatsheets/gguf-filenames.md` — decode any GGUF filename + pick a quant
- `cheatsheets/inspect-weights.md` — quick weight inspection one-liners

## Sources & videos

- `docs/references.md` — official docs & tools
- `videos.md` — curated videos
