# Cheatsheet — Downloading from the Hub

**TL;DR.** `hf download <repo> [file]` — that's 90% of it. Files land in the HF cache (resumable, deduped); `--local-dir` gives you a plain folder. Gated repos need `hf auth login` first. `--dry-run` before every big download.

## Core commands (verified 2026-10-27)

```bash
# single file (prints local path on last line)
hf download Qwen/Qwen3-8B-GGUF Qwen3-8B-Q4_K_M.gguf

# whole repo
hf download Qwen/Qwen3-8B-GGUF

# only the quants you want
hf download unsloth/Qwen2.5-VL-7B-Instruct-GGUF --include "*Q4_K_M.gguf" --include "mmproj-*.gguf"

# plain folder instead of cache (creates .cache/huggingface metadata inside)
hf download Qwen/Qwen3-8B-GGUF --local-dir ./models/qwen3-8b

# specific revision / tag / branch
hf download <repo> --revision v1.1

# see what would be downloaded (sizes, what's already cached)
hf download <repo> --dry-run

# custom cache location
hf download <repo> --cache-dir /big/disk/hf-cache

# quiet (path only — for scripts)
hf download <repo> --quiet

# hf:// URIs (type + revision + path in one string)
hf download hf://datasets/bigcode/the-stack@v1.1
hf download hf://Qwen/Qwen3-8B-GGUF@main/Qwen3-8B-Q4_K_M.gguf
```

## Python

```python
from huggingface_hub import hf_hub_download, snapshot_download

p = hf_hub_download("unsloth/Qwen2.5-VL-7B-Instruct-GGUF", "Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf")
snapshot_download("Qwen/Qwen3-8B-GGUF", local_dir="./qwen3-8b", allow_patterns=["*Q4_K_M.gguf"])
```

## Gated repos (Qwen/DeepSeek/Moonshot GGUFs, Llama)

```bash
# 1. accept terms in the browser on the repo page
# 2. log in (browser flow, or token)
hf auth login
hf auth login --token $HF_TOKEN --add-to-git-credential   # CI / scripted
# 3. download as normal
hf download Qwen/Qwen3-8B-GGUF
```

Without a token the API returns `401 Invalid username or password` — that's "gated or private", not a bug.

## Big/slow files

```bash
# aria2c: 8 parallel connections, resumable
aria2c -x 8 -s 8 -c https://huggingface.co/<org>/<model>/resolve/main/<file>.gguf
# gated: append ?token=hf_...

# slow-link timeout fix
export HF_HUB_DOWNLOAD_TIMEOUT=30
```

## Cache management

```bash
hf cache ls                                   # what's cached + sizes
hf cache ls --filter "size>10g" --revisions
hf cache rm model/<org>/<model> -y            # drop a repo
hf cache rm hf://models/<org>/<model>/<file>.gguf -y   # drop one file
hf cache prune                                # detached revisions + .incomplete
hf cache verify <repo>                        # checksums vs Hub
```

## Env vars

| Var | Purpose |
|---|---|
| `HF_TOKEN` | token (CI, scripts) |
| `HF_HOME` | cache location (default `~/.cache/huggingface`) |
| `HF_HUB_OFFLINE=1` | no network |
| `HF_HUB_DOWNLOAD_TIMEOUT` | default 10 s |
| `HF_ENDPOINT` | mirror (e.g. `https://hf-mirror.com`) |

## git lfs (legacy)

```bash
git lfs install && git clone https://huggingface.co/<org>/<model>
```

Use only when you need history/branches; `hf download` is faster.

## Verified on

- 2026-10-27, `hf` CLI (huggingface_hub 2026 release); 401 behavior on gated Qwen/DeepSeek/Moonshot GGUF repos confirmed live.
