# System prompts & chat templates

**Summary.** A chat model is just a text-completion model wearing a costume. The "chat" is a plain token sequence formatted by a **chat template** — a string pattern with special tokens (like `<|user|>`, `<|assistant|>`, `[/NO NEWLINE]`) that marks who is speaking. The **system prompt** is the first message in that sequence: standing instructions that shape the model's behavior for the whole conversation. Get the template wrong and the model ignores instructions, rambles, or never stops; get the system prompt right and a small model behaves dramatically better.

## The key insight

From the HF chat-templating docs: *"All causal LMs, whether chat-trained or not, continue a sequence of tokens."* Chat fine-tuning trains the model on sequences formatted as messages, so it learns to:

- follow the role markers,
- answer in the assistant voice,
- **stop** when it reaches the end-of-assistant-message token.

The template is the contract between your runtime and the model. Every model family has its own:

```text
# Llama 3.1 style (simplified)
<|begin_of_text|><|start_header_id|>system<|end_header_id|>
{system prompt}
<|eot_id|><|start_header_id|>user<|end_header_id|>
{user message}
<|eot_id|><|start_header_id|>assistant<|end_header_id|>
{model answer here...}<|eot_id|>
```

```text
# Qwen3 style (simplified)
<|im_start|>system
{system prompt}<|im_end|>
<|im_start|>user
{user message}<|im_end|>
<|im_start|>assistant
{model answer here...}<|im_end|>
```

```text
# ChatML (GPT / Mistral style)
system
{system prompt}
user
{user message}
assistant
{model answer here...}
```

In Hugging Face the template lives in `tokenizer_config.json` as a Jinja string (`chat_template`); in GGUF it's stored in the file metadata. **Your runtime must apply the model's own template** — llama.cpp loads it from the GGUF, Ollama/LM Studio ship it, and `transformers` applies it via `apply_chat_template()`.

## System prompts

The system message is *part of the context window* (it costs tokens every turn) and is the strongest lever you have for behavior:

- **What to put**: role ("You are a concise coding assistant"), format constraints (JSON, markdown, length), safety/behavior rules, few-shot examples if needed.
- **What it does**: steers the distribution the model samples from. Small models (≤8B) are *very* sensitive to it — a clear, specific system prompt often beats a bigger model with a vague one.
- **What it doesn't do**: it's not a hard constraint. The model can (and will) ignore it for novel situations; it's strong prior, not a rule engine.
- **Token budget**: keep it tight. A 2K-token system prompt permanently eats 2K of your context and adds prefill time every request.

## Common failure modes (all template-related)

| Symptom | Usual cause |
|---|---|
| Model answers as "user" or in mixed roles | wrong/missing chat template |
| Output never stops / repeats forever | missing end-of-turn token in template |
| Ignores instructions, acts like a base model | template doesn't match the model family |
| Weird first token / garbage | template applied twice, or raw text fed to a chat model |
| Different quality in Ollama vs llama.cpp vs HF | each applies a slightly different template version |

**Rule**: never hand-write role markers. Let the runtime apply the model's shipped template. If you must inspect it: `llama-template` (llama.cpp), `tokenizer.apply_chat_template()` (HF), or the GGUF metadata.

## Practical rules

1. **One system prompt per task**, stored as a file, versioned. A/coding vs chat vs RAG prompts are different tools.
2. **Be specific and short**: "Reply in under 100 words. Use markdown code blocks." beats a page of personality.
3. **Test the template**: run one known prompt through your runtime and check the rendered token sequence (most runtimes have a `--display-prompt` or equivalent).
4. **Don't fight the model**: if the model's default behavior is fine, don't spend tokens re-specifying it.
5. **System prompt ≠ jailbreak shield**: it shapes behavior, it doesn't enforce policy (see topic 09 on safety).

## My experiments

- (fill in: e.g. "rendered the same 3-message conversation with llama-template vs transformers apply_chat_template — identical except for one special token; Qwen3 stopped cleanly with the GGUF template, rambled with the hand-written one")
