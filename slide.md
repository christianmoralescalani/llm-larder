# slide.md — presentation plan: "Running LLMs locally"

> How the talk is structured and prepared. This file is the **plan**, not the deck —
> each slide below describes what goes on it, what to say, and what to show live.

**Audience:** technical people who want to learn about running LLMs locally.
**Format:** speaking-first + live repo/server demo. ~15 min talk + ~7 min demos.
**Slide count:** 10 — deliberately minimal. The repo is the visual aid.

## Principles

1. **Few slides, long talk.** One slide = one chapter = one demo. If a slide needs more than 5 bullets, that content belongs in the repo, not the deck.
2. **Show, don't tell.** Every claim is backed by something live: the repo, a running server, or a command.
3. **The repo is the slide deck.** When in doubt, navigate the repo instead of adding a slide.
4. **One thread.** The talk follows the life of a single model: *understand it → check it fits → download it → run it → tune it → use it → go beyond text*.

## Slide → repo map

| Slide | Folder(s) |
|---|---|
| 1. Title | (root) |
| 2. Why local | — |
| 3. Four concepts | `00-foundations/` |
| 4. The formula | `02-hardware-and-vram/` |
| 5. Getting a model | `01-huggingface/` |
| 6. Choosing a runtime | `03-ollama/`, `04-llama-cpp/`, `05-vllm/` |
| 7. Live: docker run → chat | `04-llama-cpp/`, `08-apis-and-clients/` |
| 8. The dials | `06-sampling-and-prompting/` |
| 9. Beyond text | `07-multimodal-vision/` |
| 10. Use the repo + Q&A | (root), `09-glossary-and-links/` |

---

## Slides

### 1. Title — "LocalLLM: running LLMs locally"

**On the slide.** Title + one-liner: *"Notes, cheatsheets, and a working setup for running LLMs on your own hardware."* Speaker name + repo link. Nothing else.

**Say (30 s).** What the repo is: a knowledge base built around *actually running* models locally — not theory. Everything in it has been run on real hardware.

**Demo.** None. (Optionally have the repo root open in a browser as the backdrop.)

### 2. Why local

**On the slide.** Four bullets, one line each:
- **Privacy** — data never leaves the machine
- **Cost** — no subscription, no per-token billing
- **Offline** — works on a plane, in a cabin, anywhere
- **Control** — your hardware, your parameters, your model

**Say (45 s).** The pitch, not the argument — a technical audience already believes it. Land on *control*: with a local model you can change anything, and this repo documents what changing things actually does.

**Demo.** None.

### 3. The four concepts that explain everything

**On the slide.** Four terms, one line each (from `00-foundations/`):
- **Token** — the unit of everything: text, context, speed, cost
- **Parameter** — what "7B" / "27B" means: size of the model
- **Quantization** — compressing weights (Q4_K_M, Q8_0…) so big models fit
- **KV cache** — the memory that grows with context, and eats VRAM

**Say (2 min).** These four words carry the whole talk. Everything else — VRAM math, speed, which model to pick — is arithmetic on them. No deep theory; the repo has the full notes if anyone wants to go deeper.

**Demo.** Open `00-foundations/Index.md` — "this is how the repo teaches it: start here, then in order."

### 4. The formula: will it fit?

**On the slide.** One equation + one worked example:

```
VRAM needed = weights (params × bytes-per-weight) + KV cache (context) + overhead
```

Worked: *Qwen3.8-27B @ Q4_K_XL ≈ 16 GB weights + ~1–2 GB KV @ 32k ctx → runs on a 16–24 GB card.*

**Say (2 min).** This is the only formula you need. If it fits in VRAM, it's fast; if it doesn't, you hit the CPU-offload speed cliff. The quant changes the first term, the context length changes the second.

**Demo.** Open `02-hardware-and-vram/cheatsheets/vram-budget.md` — *"I have X GB → what fits"* — the table the audience can steal.

### 5. Getting a model

**On the slide.** Three steps, three commands:
1. **Search** — task + size + quant (e.g. "27B Q4 GGUF instruct")
2. **Inspect** — files, quant, context, model card — *before* the 15 GB download
3. **Download** — `hf` CLI, single file, resumable

**Say (90 s).** The Hub is the Docker Hub for models. GGUF is the local format. The three-step habit — search → inspect → download — saves wasted bandwidth and wrong-model days.

**Demo.** Run (or paste) a real `hf` search + download command from `01-huggingface/cheatsheets/hf-download.md`.

### 6. Choosing a runtime

**On the slide.** Three columns:

| | Ollama | llama.cpp | vLLM |
|---|---|---|---|
| One-liner | fastest to start | most control | max throughput |
| Pick it when | personal use, few users | you want every flag, GGUF, vision | many concurrent users, serving |

**Say (2 min).** All three speak the *same* OpenAI-compatible API — so your client code doesn't change when you switch (foreshadows slide 7). Pick by use case, not by religion.

**Demo.** Open `05-vllm/cheatsheets/vllm-vs-llamacpp-vs-ollama.md` — the full decision table.

### 7. Live: from `docker run` to chat

**On the slide.** The real command (from `testcommands`), abridged:

```
docker run -d --gpus all -p 8080:8080 -v ./models:/models \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  -m /models/Qwen3.8-27B-UD-Q4_K_XL.gguf --mmproj ... \
  --reasoning on -c 131072 --flash-attn --temp 0.6 --top-p 0.95 --top-k 20
```

**Say (3 min).** This is the actual setup, not a toy. Walk three flags: the **quant** (why it fits — slide 4), the **context** (why KV cache matters — slide 3), **`--reasoning on`** (it's a thinking model — slide 8). Then the payoff: one API for everything — `curl http://localhost:8080/v1/chat/completions` works the same against Ollama, llama.cpp, and vLLM.

**Demo (the centerpiece, ~4 min).** Show the running server → `curl` a chat completion → open the web UI in a browser → ask it something. If the live server is down, fall back to the repo: `04-llama-cpp/cheatsheets/llama-cpp-docker.md` has the exact command.

### 8. The dials

**On the slide.** The knobs, one line each (from `06-sampling-and-prompting/`):
- `temperature` — randomness (lower = more focused)
- `top_p` / `top_k` / `min_p` — how the candidate set is cut
- **Reasoning budget** — thinking on/off/limited for reasoning models

**Say (2 min).** Sampling is the quality dial. The values on slide 7 (`0.6 / 0.95 / 20`) are working defaults, not magic — change them per task. For thinking models, the budget is the new knob: more thinking = better but slower.

**Demo.** Same live server: re-run the prompt with a different `temperature` and show the difference. 30 seconds, high impact.

### 9. Beyond text

**On the slide.** Three lines (from `07-multimodal-vision/`):
- **Vision** — image → visual tokens; an image *costs* context (the `--mmproj` flag from slide 7)
- **Embeddings** — the RAG building block
- **Audio** — Whisper-family transcription

**Say (90 s).** Same runtimes, same API, same formula — just different model files. The only new concept is that images consume tokens, so vision needs VRAM headroom.

**Demo.** Send an image to the running server (it already has `--mmproj` loaded) — the model describes it.

### 10. Use the repo + Q&A

**On the slide.**
- 10 folders × 5 content types: `notes/` · `cheatsheets/` · `docs/` · `videos.md` · `Index.md`
- **"New to a topic? Open its `Index.md` — it says what to read first and in what order."**
- Repo link · `09-glossary-and-links/` for terms + video master list

**Say (60 s).** How to keep learning after the talk: every folder has an index with a reading order, cheatsheets for the daily work, and a curated video list. The glossary is living — terms get added as they come up.

**Demo.** Navigate the repo live: open an `Index.md`, click through its reading order, land on a cheatsheet. End on the repo root.

---

## Prep checklist (before the talk)

- [ ] Repo cloned and open in a browser on the demo machine (GitHub view, so links work)
- [ ] Model downloaded to `./models/` (Qwen3.8-27B-UD-Q4_K_XL + mmproj)
- [ ] Server started **before** the talk (`docker run` from slide 7) — verify with `curl localhost:8080/v1/models`
- [ ] Slide 8 demo prompt prepared (one short prompt, two temperatures)
- [ ] Slide 9 demo image prepared (one clear photo, < 1 MB)
- [ ] `curl` one-liners saved in a scratch file (don't type them live)
- [ ] Fallback plan: if the server dies mid-talk, the repo cheatsheets are the backup — the talk still works, just less live
- [ ] Internet check: the repo browser view needs it; if offline, use the local clone in a markdown viewer

## Timing

| Segment | Slides | Time |
|---|---|---|
| Setup | 1–2 | ~1.5 min |
| Concepts | 3–4 | ~4 min |
| Model + runtime | 5–6 | ~3 min |
| Live demo | 7 | ~7 min (incl. demos) |
| Tuning + beyond | 8–9 | ~3 min |
| Close | 10 | ~1.5 min |
| **Total** | 10 slides | **~20 min + Q&A** |
