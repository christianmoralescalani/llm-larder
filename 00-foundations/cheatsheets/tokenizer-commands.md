# Cheatsheet — Tokenizer commands

**TL;DR.** Always count tokens with the *model's own* tokenizer. OpenAI models → `tiktoken`. Everything else → the Hugging Face tokenizer (`transformers` or the lightweight `tokenizers`). English ≈ 1.0–1.3 tokens/word; code/math/emoji cost more.

## tiktoken (OpenAI models)

```bash
pip install tiktoken
```

```python
import tiktoken

enc = tiktoken.get_encoding("o200k_base")   # GPT-4o / o-series
# enc = tiktoken.get_encoding("cl100k_base") # GPT-3.5 / GPT-4 / GPT-4-turbo

text = "Hello, how are you today?"
ids = enc.encode(text)
print(len(ids))                          # token count
print([enc.decode([i]) for i in ids])    # see the pieces
```

| Encoding | Used by | Vocab |
|---|---|---|
| `cl100k_base` | GPT-3.5, GPT-4, GPT-4-turbo | 100,277 |
| `o200k_base` | GPT-4o, o1/o3 | 200,019 |
| `p50k_base` | GPT-3 (legacy) | 50,257 |

## Hugging Face tokenizer (all other models)

```bash
pip install transformers        # full
# or: pip install tokenizers huggingface_hub  # lightweight, no torch needed
```

```python
from transformers import AutoTokenizer

tok = AutoTokenizer.from_pretrained("Qwen/Qwen3-8B")
ids = tok.encode("Hello, how are you today?")
print(len(ids))
print([tok.decode([i]) for i in ids])

# whole conversation, the way the model sees it:
msgs = [{"role": "user", "content": "Hi!"},
        {"role": "assistant", "content": "Hello!"},
        {"role": "user", "content": "Count tokens."}]
print(len(tok.apply_chat_template(msgs, add_generation_prompt=True)))
```

Lightweight variant (no torch, works in a venv):

```python
from tokenizers import Tokenizer
tok = Tokenizer.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")
print(tok.get_vocab_size())
print(len(tok.encode("hello world").ids))
```

## GGUF (llama.cpp)

```bash
llama-tokenizer -m model.gguf "some text"        # tokenize with the GGUF's own tokenizer
llama-template -m model.gguf -p "prompt"         # show the rendered chat template
```

## Rule of thumb table

| Text type | Tokens per word (approx) |
|---|---|
| English prose | 1.0–1.3 |
| Code | 1.5–3 |
| Math / symbols | 3–10 |
| Non-Latin (CJK) | 1–2 per *character* (no word concept) |
| Emoji | 1–4 each |

## Verified on

- tiktoken 0.14.0, `tokenizers` (HF), run 2026-10-27.
- Sample outputs: "Hello, how are you today?" → 7 tokens (cl100k_base); "2 + 2 = 4" → 7 tokens; "🤗" → 3 tokens; 90-word paragraph → 101 tokens (≈1.12/word).
