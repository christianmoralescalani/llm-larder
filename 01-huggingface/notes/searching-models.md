# Searching models on the Hugging Face Hub

**Summary.** The Hub (huggingface.co) is where almost every local model lives. You can search it three ways: the **web UI** (URLs are filter recipes you can copy-paste), the **`hf models ls` CLI**, and the **search API** (`/api/models`) for scripting. A repo is `org/model` with tags (task, library, license, base model) that drive all the filters. Before downloading anything, spend 2 minutes on the model card: official vs community author, license, context length, and what the numbers actually mean.

## How the Hub is organized

- **Repo id**: `org/model` (or just `user/model`). Examples: `Qwen/Qwen3-8B`, `unsloth/Qwen2.5-VL-7B-Instruct-GGUF`.
- **Tags** (all filterable):
  - `pipeline_tag` — task: `text-generation`, `image-text-to-text` (vision), `feature-extraction` (embeddings), `audio-classification`…
  - `library` — `gguf`, `transformers`, `vllm`, `mlx`…
  - `license:*` — `apache-2.0`, `mit`, `llama3.1`, `qwen`, `deepseek-research`, `cc-by-nc-4.0`, `other`…
  - `base_model:*` / `base_model:quantized:*` — which full-precision model a quantized repo came from.
  - `gated` — requires login + accepting terms (see below).
- **Metrics**: downloads, likes, last updated. Downloads are a decent "someone actually ran this" proxy, not a quality measure.

## Web search (URL recipes)

Base URL: `https://huggingface.co/models` — every filter is a query parameter:

| Param | Values | Meaning |
|---|---|---|
| `search=` | free text | matches repo id |
| `sort=` | `trending`, `downloads`, `likes`, `createdAt`, `lastModified` | ordering |
| `filter=` | comma list of pipeline tags, `gguf`/`transformers`/…, `apache-2.0`/… | AND-combined |
| `author=` | org/user | only that author's repos |

Copy-paste recipes (all verified working 2026-10-27):

```text
# Best 8B GGUF chat models by downloads
https://huggingface.co/models?search=8B&filter=gguf&sort=downloads

# Apache-2.0 GGUF text-generation (commercially safe)
https://huggingface.co/models?filter=text-generation,gguf,apache-2.0&sort=downloads

# Vision models in GGUF
https://huggingface.co/models?filter=image-text-to-text,gguf&sort=downloads

# All official Qwen GGUF repos
https://huggingface.co/models?author=Qwen&search=GGUF

# Embedding models for RAG
https://huggingface.co/models?filter=feature-extraction,gguf&sort=downloads
```

Tip: click any filter chip in the web UI and read the resulting URL — that's the recipe.

## `hf models` CLI

```bash
hf models ls --search "8b" --sort downloads --limit 10
hf models ls --author Qwen --pipeline-tag text-generation
hf models ls --num-parameters min:6B,max:128B      # by parameter count
hf models ls --apps llama.cpp                       # runnable by llama.cpp
hf models ls --no-gated --author google             # skip gated repos
hf models ls --search bert --limit 2 --json | jq '.[].id'

# One repo: files, info, card
hf models ls meta-llama/Llama-3.2-1B-Instruct -R -h      # file tree w/ sizes
hf models info Qwen/Qwen3-8B
hf models card Qwen/Qwen3-8B --metadata                  # YAML frontmatter only
```

## Search API (for scripts)

```bash
# list ids
curl -s "https://huggingface.co/api/models?search=8B&filter=gguf&sort=downloads&direction=-1&limit=10" \
  | jq -r '.[].modelId'

# with metrics
curl -s "https://huggingface.co/api/models?search=Qwen3&filter=gguf&sort=downloads&limit=5&expand[]=downloads&expand[]=likes" \
  | jq '.[] | {id: .modelId, downloads, likes}'

# single repo: files + sizes + tags
curl -s "https://huggingface.co/api/models/unsloth/Qwen2.5-VL-7B-Instruct-GGUF?blobs=true" \
  | jq '.siblings[].rfilename'

# parameter count (safetensors repos only)
curl -s "https://huggingface.co/api/models/Qwen/Qwen3-8B?expand[]=safetensors" \
  | jq '.safetensors'
# → {"parameters": {"BF16": 8190735360}, "total": 8190735360}
```

Full spec: `https://huggingface.co/.well-known/openapi.json` (or `.md`). Datasets: `/api/datasets`, Spaces: `/api/spaces`.

## Evaluating a repo before downloading

Checklist, in order:

1. **Who published it?** Official orgs (`Qwen`, `meta-llama`, `google`, `deepseek-ai`, `moonshotai`, `ggml-org`) = reference weights. Community quants: `bartowski`, `unsloth`, `mradermacher`, `lmstudio-community` (see `formats-and-quantization.md`).
2. **License tag** — what you're allowed to do (below).
3. **Model card** — architecture, context length, recommended sampling, benchmarks, known issues. `hf models card <id> --text` fetches it.
4. **Downloads/likes + recency** — a 5 GB quant nobody downloaded is riskier than a popular one.
5. **`base_model:quantized:` tag** — confirms which checkpoint it was quantized from.
6. **Discussions tab** — known problems, "garbled output on X" reports.
7. **Gated?** Qwen, DeepSeek and Moonshot GGUF repos (and most Llama repos) require login + accepting terms; without a token the API returns `401`.

### Reading benchmarks honestly

Model-card benchmarks are self-reported, cherry-picked, and often measured on the model's own sweet spot. Two models can swap places depending on the task. If it matters to you, run 5–10 of your own prompts on the finalists (cheap: `llama-cli` or an OpenAI-compatible endpoint).

## Licenses — what you can and can't do

| License | Typical models | What it means for you |
|---|---|---|
| **MIT** | SmolLM, some open releases | Do anything, keep the license text. |
| **Apache-2.0** | Qwen, Gemma 3 (with extra terms), Mistral small | Do anything incl. commercial; patent grant; keep notices. |
| **Llama Community License** (3.1/3.2/3.3, `llama3.x` tag) | Meta Llama | Free incl. commercial, **except** companies with >700M monthly users (they must ask Meta). Fine for personal/local use. |
| **Qwen license** | Qwen base models | Free incl. commercial with usage-policy conditions (e.g. no illegal use); read the card. |
| **DeepSeek license** | DeepSeek-V3/R1 | MIT for code; weights under a research-leaning license — check the card before commercial use. |
| **`cc-by-nc-4.0` / research-only** | some academic releases | **No commercial use.** Fine for learning, not for a product. |
| **`other`** | Kimi K2, some MoE | Custom terms — read the LICENSE file in the repo. |

Rule of thumb: for anything you might ship, prefer `apache-2.0` / `mit`; for personal local use, Llama Community and Qwen licenses are fine.

## My experiments

- (fill in: e.g. "searched `filter=image-text-to-text,gguf` for a 12B vision model; unsloth Qwen2.5-VL-7B Q4_K_M + mmproj-F16 worked first try with llama-server")
