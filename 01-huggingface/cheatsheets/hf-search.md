# Cheatsheet — Searching models on the Hub

**TL;DR.** Web URL = fastest; `hf models ls` = terminal; `/api/models` = scripts. Filters are comma-joined in `filter=`: pipeline tag + `gguf` + license. Sort by `downloads` to find what people actually run.

## Web URL recipes (verified 2026-10-27)

| Goal | URL |
|---|---|
| Best 8B GGUF by downloads | `https://huggingface.co/models?search=8B&filter=gguf&sort=downloads` |
| Apache-2.0 GGUF text-gen | `https://huggingface.co/models?filter=text-generation,gguf,apache-2.0&sort=downloads` |
| Vision GGUF | `https://huggingface.co/models?filter=image-text-to-text,gguf&sort=downloads` |
| Embedding GGUF (RAG) | `https://huggingface.co/models?filter=feature-extraction,gguf&sort=downloads` |
| One author's GGUFs | `https://huggingface.co/models?author=Qwen&search=GGUF` |
| Trending this week | `https://huggingface.co/models?sort=trending` |
| Official ggml-org quants | `https://huggingface.co/ggml-org` |

URL params: `search=` (text) · `sort=` (`trending|downloads|likes|createdAt|lastModified`) · `filter=` (pipeline tag, library, license — AND) · `author=`.

## `hf models ls`

```bash
hf models ls --search "8b" --sort downloads --limit 10
hf models ls --author Qwen --pipeline-tag text-generation
hf models ls --num-parameters min:6B,max:128B
hf models ls --apps llama.cpp --no-gated
hf models ls --search bert --limit 2 --json | jq '.[].id'
hf models info Qwen/Qwen3-8B
hf models card Qwen/Qwen3-8B --metadata     # YAML frontmatter (license, tags)
```

## Search API (curl + jq)

```bash
# ids
curl -s "https://huggingface.co/api/models?search=8B&filter=gguf&sort=downloads&direction=-1&limit=10" | jq -r '.[].modelId'

# ids + metrics
curl -s "https://huggingface.co/api/models?search=Qwen3&filter=gguf&sort=downloads&limit=5&expand[]=downloads&expand[]=likes" \
  | jq -r '.[] | "\(.modelId)\t\(.downloads)\t\(.likes)"'

# repo files + sizes
curl -s "https://huggingface.co/api/models/<org>/<model>?blobs=true" | jq -r '.siblings[] | "\(.size)\t\(.rfilename)"'

# param count (safetensors repos)
curl -s "https://huggingface.co/api/models/Qwen/Qwen3-8B?expand[]=safetensors" | jq '.safetensors.total'
```

## 30-second repo evaluation

1. Author: official org or known quant provider? (`bartowski`, `unsloth`, `mradermacher`, `lmstudio-community`, `ggml-org`)
2. `license:*` tag → allowed use (see `notes/searching-models.md` table)
3. `base_model:quantized:*` → which checkpoint it came from
4. Downloads/likes + recency
5. Model card: context, recommended sampling, known issues
6. Gated? → `hf auth login` first

## Verified on

- 2026-10-27, all URL recipes and API calls executed live; `hf` CLI from `huggingface_hub` (2026 release).
