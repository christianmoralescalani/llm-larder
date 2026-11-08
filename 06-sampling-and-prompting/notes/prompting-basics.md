# Prompting basics for local models

**Summary.** Local models (≤8B, especially quantized) are less forgiving than frontier models: weaker instruction-following, less tolerance for ambiguity, and a weaker ability to infer what you *meant*. The fix is not a bigger model — it's **clearer, more specific, more constrained prompts**: tight system prompts (role + format + limits), few-shot examples for format tasks, and structured-output enforcement (JSON schema / grammar) instead of "please output JSON". The system prompt is the strongest behavior lever you have, and it costs tokens every turn — keep it short.

## Why local models need clearer instructions

- **Less robustness to ambiguity** — a frontier model guesses your intent; a 7B model does the literal, boring thing (or something random). State the intent explicitly.
- **Weaker implicit knowledge** — don't rely on "you know what I mean" conventions. Spell out format, length, and edge cases.
- **Quantization degrades instruction-following first** — Q4 models drop subtle instruction adherence before they drop factual knowledge. If a prompt works at Q8 but not Q4, tighten the prompt, don't just re-quant.
- **Smaller context discipline** — small models lose the thread in long contexts; put critical instructions *near* the question, not only in a 2k-token system prompt.
- **They follow the template, not your vibes** — the chat template + system prompt define the "mode" the model operates in; vague prompts leave it in base-model mode.

## System prompt design

Anatomy of a tight system prompt (order matters — model reads top to bottom):

```text
1. ROLE      — who it is / what it does (one line)
2. TASK      — what it should do, in active voice
3. FORMAT    — exact output shape (JSON keys, markdown, language, length)
4. CONSTRAINTS — hard rules (max length, what to refuse, what to never do)
5. EXAMPLES  — 1–3 few-shot pairs (only if format/style is the point)
```

Example (code reviewer):

```text
You are a senior code reviewer.
Review the code the user provides.
Output format: markdown. Start with a one-line verdict (OK / CHANGES NEEDED),
then at most 3 bullets, each citing the line number.
Do not rewrite the code. Do not discuss style beyond the cited lines.
```

Rules:

1. **Short beats long.** Every token in the system prompt is paid on *every* request (prefill time + context). A 2k-token system prompt is a 2k-token tax.
2. **One prompt per task**, stored as a file, versioned. Chat-prompt ≠ RAG-prompt ≠ JSON-prompt.
3. **Positive phrasing** — "Answer with only the JSON object" beats "Don't add commentary, don't use markdown, don't explain".
4. **Be specific about format** — "JSON with keys `answer` (string) and `confidence` (0-1)" beats "return JSON".
5. **Don't re-specify what the model already does** — if the default behavior is fine, don't spend tokens on it.
6. **Mind the model's built-in system prompt** — Llama 3.x ships with one (safety/identity); yours is *added* to it, and the two can conflict.

## Few-shot examples

Give 1–4 input→output pairs to pin down a format or style. Works best for: classification, extraction, fixed-format generation, tone matching.

```text
Classify the sentiment. Answer with exactly one word: positive, negative, or neutral.

"the build finally passed" → positive
"another regression in CI" → negative
"the meeting moved to 3pm" → neutral
"your sentence here" →
```

- **2–4 examples is usually the sweet spot** — more costs context with diminishing returns on small models.
- Put examples in the **system prompt** (stable, cached by prefix caching) or as `MESSAGE` pairs in an Ollama Modelfile — not scattered in user messages.
- **Order matters**: put the most representative example last (recency bias).
- **Watch token cost**: each example is context on every turn. For long-running chats, few-shot in the system prompt is a real budget line.
- Few-shot is a *format* tool, not a *knowledge* tool — examples don't teach the model new facts.

## Structured output (make it impossible to break)

"Please respond in JSON" is a request; **constrained decoding** is a guarantee. Ladder of enforcement, weakest → strongest:

1. **Prompting** — "Return only JSON with keys …" — works ~90–99% of the time; validate and retry on failure.
2. **JSON mode** — Ollama: `"format": "json"` in the API; llama.cpp: `--json-schema` / `-jf` (JSON-schema constraint); vLLM: `guided_json` / `response_format`. Output is *guaranteed* valid JSON.
3. **Schema-constrained** — supply the actual JSON schema (or a BNF grammar in llama.cpp); keys, types, enums, and value ranges are enforced token-by-token.
4. **Post-hoc validation + retry** — whatever the method, validate against the schema and retry with the error in the prompt.

```bash
# llama.cpp server: JSON-schema constraint (server-level)
# … --json-schema '{"type":"object","properties":{"answer":{"type":"string"}},"required":["answer"]}'

# Ollama API: JSON mode
curl http://localhost:11434/api/generate -d '{
  "model": "qwen3:8b",
  "prompt": "Extract the city from: \"I live in Lisbon.\"",
  "format": "json"
}'
```

For reasoning models: thinking tokens come *before* the JSON — use the runtime's reasoning parser so `content` is clean JSON, or turn thinking off for structured pipelines.

## Prompt patterns that pay off locally

| Pattern | How | Why it helps locally |
|---|---|---|
| **Delimiters for RAG** | wrap context: `Context:\n\"\"\"…\"\"\"\nQuestion: …` | small models bleed context into the answer without clear boundaries |
| **Restate the task at the end** | put the actual question *after* the context | recency — the last tokens before generation get the most attention |
| **Length caps** | "Answer in under 100 words" | small models ramble; caps also cap your decode time |
| **Step-by-step for hard tasks** | "Think step by step, then answer" | manual CoT — but for reasoning models, just turn thinking on instead |
| **"If you don't know, say so"** | explicit fallback instruction | small models hallucinate confidently; the fallback line measurably reduces it |
| **One question per turn** | split multi-part asks | multi-part prompts are where small models drop sub-questions |

## Common mistakes

1. **Vague system prompt + expecting frontier behavior** — "You are a helpful assistant" does almost nothing for a 7B model.
2. **Burying the instruction** — 500 words of context, then the actual ask. Restate it at the end.
3. **Negative instructions** — "don't mention X" makes X more likely; phrase positively.
4. **Few-shot bloat** — 10 examples eating 1.5k tokens of a 4k context.
5. **Prompting for structure instead of constraining it** — if the output feeds code, use a schema, not a request.
6. **Forgetting prefix caching** — volatile content (timestamps, random IDs) at the *start* of your prompt invalidates the cache every turn; put it at the end.
7. **Testing on one example** — A/B prompt changes on a fixed set of 10–20 prompts with a fixed `seed`, or you're chasing noise.

## Sources

- [OpenAI — Prompt engineering guide (patterns, few-shot, CoT)](https://platform.openai.com/docs/guides/prompt-engineering)
- [Anthropic — Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [HF — Chat templating (why format is everything)](https://huggingface.co/docs/transformers/main/en/chat_templating)
- [This repo — 00-foundations system prompts & chat templates](../../00-foundations/notes/system-prompts-and-chat-templates.md)
- [This repo — 06 cheatsheets/task-settings.md (task → settings presets)](../cheatsheets/task-settings.md)

## My experiments

- (fill in: e.g. "RAG on Qwen3.8-27B: adding `Context:\n\"\"\"…\"\"\"` delimiters + restating the question last cut 'context bleed' failures from ~1 in 10 to 0 in 50; JSON schema constraint vs plain 'return JSON': 0/50 vs 2/50 parse failures; 3-shot sentiment classifier at 8B hit 94% — 4th example added nothing")
