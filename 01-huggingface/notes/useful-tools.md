# Extra useful tools for Hub work

**Summary.** Beyond "download a model", the Hub tooling covers auth, cache management, repo management, uploading your own quants, dataset leaderboards, and scripting via the API. The `hf` CLI (successor of `huggingface-cli`) does 90% of it; for the rest there's `huggingface_hub` (Python), `git lfs`, `aria2c`, `hf-mirror.com`, and the `transformers` quick-test path.

## The `hf` CLI (current, verified 2026-10-27)

Install: `curl -LsSf https://hf.co/cli/install.sh | bash` (or `pip install -U huggingface_hub`, `brew install hf`, or `uvx hf …` for one-offs). `hf update` to upgrade.

| Group | Commands | Notes |
|---|---|---|
| **auth** | `hf auth login` (browser flow or `--token $HF_TOKEN --add-to-git-credential`), `hf auth whoami`, `hf auth logout` | tokens live in `~/.cache/huggingface/token`; `HF_TOKEN` env var overrides |
| **download** | `hf download <repo> [files…]` | `--include/--exclude` patterns, `--revision`, `--local-dir`, `--dry-run`, `--cache-dir`, `--quiet`, `hf://` URIs, `--repo-type dataset/space` |
| **models** | `hf models ls/info/card` | search, `--num-parameters min:6B,max:128B`, `--apps llama.cpp`, `--no-gated`; `card --metadata/--text` |
| **cache** | `hf cache ls/rm/prune/verify` | `--filter "size>30g"`, per-file removal via `hf://` URIs, checksum verification |
| **repos** | `hf repos ls/create/delete-files`, `hf repos branch/tag` | manage your own repos |
| **upload** | `hf upload <repo> <path> [path-in-repo]` | publish your own GGUFs; `--create-pr`, `--every` for periodic sync |
| **cp** | `hf cp <src> <dst>` | local ↔ Hub, stdin/stdout with `-`, repo→repo copies |
| **datasets** | `hf datasets ls/parquet/sql/leaderboard` | `leaderboard SWE-bench/SWE-bench_Verified` = model scores on a benchmark; `sql` runs DuckDB on dataset parquet |
| **misc** | `hf env`, `hf version`, `hf papers`, `hf spaces`, `hf jobs`, `hf buckets` | `hf env` output is what you paste into bug reports |

Output flags on nearly everything: `--json` (pipe to `jq`), `-q` (IDs only), `--no-truncate`.

```bash
# handy combos
hf models ls --search "8b" --sort downloads -q | head          # ids for scripting
hf download <repo> --include "*Q4_K_M.gguf" --local-dir ./models   # one quant, plain folder
hf cache rm $(hf cache ls --filter "accessed>1y" -q) -y        # reclaim disk
hf upload my-org/my-model ./out .                              # publish a quant
```

## `huggingface_hub` (Python)

```python
from huggingface_hub import hf_hub_download, snapshot_download, HfApi

# single file (cached, resumable)
p = hf_hub_download("unsloth/Qwen2.5-VL-7B-Instruct-GGUF",
                    "Qwen2.5-VL-7B-Instruct-Q4_K_M.gguf")

# whole repo to a plain dir
snapshot_download("Qwen/Qwen3-8B-GGUF", local_dir="./qwen3-8b",
                  allow_patterns=["*Q4_K_M.gguf"])

# metadata
api = HfApi()
print(api.model_info("Qwen/Qwen3-8B", files_metadata=True).siblings)
```

Bonus: `InferenceClient` — free hosted inference for a 30-second sanity check of a model before you download 5 GB.

## `git lfs` (legacy path)

```bash
git lfs install
git clone https://huggingface.co/<org>/<model>
```

Still works, but the `hf` CLI is faster (parallel, resumable, better cache). Use git only when you need repo history/branches.

## `aria2c` for big/slow downloads

When the CDN is slow or you want parallel connections + resume on a direct file:

```bash
aria2c -x 8 -s 8 -c \
  https://huggingface.co/<org>/<model>/resolve/main/<big-file>.gguf
```

(`-c` = continue; `resolve/main/` = direct file URL. Gated repos need `?token=hf_…` appended.)

## `hf-mirror.com` (blocked regions)

If huggingface.co is unreachable (e.g. mainland China):

```bash
export HF_ENDPOINT=https://hf-mirror.com
hf download Qwen/Qwen3-8B-GGUF
```

Same API shape, mirrored CDN.

## `transformers` — quick tests, not serving

```python
from transformers import pipeline

gen = pipeline("text-generation", model="Qwen/Qwen3-8B", device="cuda")
print(gen("Explain KV cache in one sentence.", max_new_tokens=64)[0]["generated_text"])
```

Use it to sanity-check a model (does it speak? does the template work?) — for real throughput use llama.cpp/Ollama/vLLM. `AutoModelForCausalLM` + `AutoTokenizer` is the lower-level equivalent when you need control.

## llama.cpp helpers that pair with Hub work

- **`llama-fit-params`**: prints the CLI args (`-ngl`, `-c`, …) that make a model fit your free VRAM — feed them to `llama-server`/`llama-cli`. (`--fit` is now on by default in the server.)
- **`llama-bench` / `llama-perplexity`**: compare quants you downloaded (see `04-llama-cpp`).

## HF API for scripting

- Models: `GET /api/models?search=&filter=&sort=&author=&expand[]=…`
- Repo: `GET /api/models/<id>?blobs=true` (files+sizes), `…/resolve/main/<file>` (file content)
- Datasets: `/api/datasets`, Spaces: `/api/spaces`
- Full OpenAPI: `https://huggingface.co/.well-known/openapi.json` (`.md` for agents)
- Rate-limited; a token raises limits.

## Environment variables worth knowing

| Var | Purpose |
|---|---|
| `HF_TOKEN` | token without interactive login (CI, scripts) |
| `HF_HOME` | move the cache off `~/.cache/huggingface` |
| `HF_HUB_OFFLINE=1` | never touch the network (air-gapped boxes) |
| `HF_HUB_DOWNLOAD_TIMEOUT` | raise from 10 s on slow links |
| `HF_ENDPOINT` | alternative endpoint (mirror) |

## My experiments

- (fill in: e.g. "moved HF_HOME to NVMe: first download of 5 GB GGUF went 40 → 120 MB/s")
