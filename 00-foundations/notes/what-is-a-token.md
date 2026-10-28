# What is a token

**Summary.** A token is the basic unit of text an LLM reads and writes — a chunk of text between a character and a whole word. Every LLM uses *subword tokenization* (usually BPE or byte-level BPE): frequent words stay whole, rare words split into learned pieces, and any text on Earth can be represented. Tokens matter for local inference because **everything is counted in tokens, not words**: context window, KV cache size, generation speed (tokens/sec), and API pricing.

## Token ≠ word

- English text runs roughly **1.0–1.3 tokens per word**. Code, math, and non-English text can be much worse (a single emoji can be 3+ tokens; `2 + 2 = 4` is 7 tokens).
- A model never sees "words" — it sees a list of integer token IDs, each mapped to a vector (embedding). The vocabulary (the ID→text table) typically has 32K–200K entries.

## Why subword tokenization?

| Approach | Problem |
|---|---|
| Word-level | Vocabulary explodes (every inflection is a new word), and unseen words are impossible (`<unk>`) |
| Character-level | Sequences become 5–10× longer, each token carries little meaning |
| **Subword (BPE family)** | Compromise: compact vocab + every text is representable + meaningful chunks |

### BPE (Byte Pair Encoding)

The algorithm (simplified, from the HF docs):

1. Start with a base vocabulary of individual characters.
2. Count frequencies of adjacent pairs in the training corpus.
3. Merge the most frequent pair into a new vocabulary entry.
4. Repeat until the target vocabulary size is reached.

Example: `("hug",10) ("pug",5) ("pun",12) ("bun",4) ("hugs",5)` → first merge `u`+`g` (appears in hug/pug/hugs), then `u`+`n` (pun/bun), etc.

### Byte-level BPE

Instead of a base vocabulary of Unicode characters (huge), use the **256 possible byte values**. Consequence: there is **no `<unk>` token** — any byte sequence, including emoji and CJK, can be tokenized. This is what GPT-2 (vocab 50,257 = 256 + 50,000 merges + eot), Llama, Qwen and most modern models use.

### Other algorithms you'll meet

- **Unigram** (T5, Gemma): probabilistic — keeps the subwords that best explain the corpus; can pick among multiple valid segmentations.
- **WordPiece** (BERT family): like BPE but merges the pair that maximizes corpus likelihood.
- **SentencePiece**: applies BPE/Unigram to *raw bytes* (no whitespace assumption), marks spaces with `▁` — needed for CJK.

### Vocab sizes in the wild

| Tokenizer | Vocab |
|---|---|
| GPT-2 (byte-level BPE) | 50,257 |
| cl100k_base (GPT-3.5/4) | 100,277 |
| o200k_base (GPT-4o) | 200,019 |
| Llama 3 | 128,256 |
| Qwen 2.5 / 3 | 151,665 |

## Real demos (verified 2026-10-27, tiktoken 0.14.0)

```text
"Hello, how are you today?"
  → 5 words → 7 tokens: ['Hello', ',', ' how', ' are', ' you', ' today', '?']
  (note: punctuation sticks left, spaces stick right)

"tokenization"   → ['token', 'ization']          (2 tokens)
"unbelievably"   → ['un', 'belie', 'vably']      (3 tokens)
"🤗"             → 3 byte-tokens
"2 + 2 = 4"      → 7 tokens: ['2',' +',' ','2',' =',' ','4']

10× "The quick brown fox jumps over the lazy dog."
  → 90 words → 101 tokens  (≈1.12 tokens/word)

Same sentence "The model was trained on a large corpus of text data."
  → cl100k_base: 12 tokens, o200k_base: 12 tokens (varies by text)
```

With the Qwen2.5-0.5B tokenizer (`tokenizers` lib, vocab 151,665):
`"Hello, how are you today? tokenization 🤗"` → **12 tokens** — a different count than tiktoken for the same text. **Always count tokens with the model's own tokenizer.**

## Why it matters for local inference

1. **Context window is in tokens.** A "4K context" model holds ≈ 3,000 English words ≈ 2–3 pages — not 4,000 words.
2. **KV cache memory scales with token count** (see [context-window.md](context-window.md) and [../cheatsheets/kv-cache-formula.md](../cheatsheets/kv-cache-formula.md)).
3. **Speed is measured in tokens/sec** — decode speed, prefill speed (see [inference-pipeline.md](inference-pipeline.md)).
4. **API cost is per million tokens** — when comparing cloud vs local, convert.
5. **Tokenizers differ per model family.** "8K context" means different amounts of *your* text for Llama vs Qwen vs Mistral.

## Practical rules

- Count tokens with `tiktoken` for OpenAI models, or the model's own HF tokenizer for everything else (commands in [../cheatsheets/tokenizer-commands.md](../cheatsheets/tokenizer-commands.md)).
- Budget: system prompt + history + answer must all fit in the context window.
- Long prompts = more prefill time + more KV cache. Don't paste 100K tokens "just in case".

## My experiments

- 2026-10-27: ran tiktoken 0.14.0 (`cl100k_base`, `o200k_base`) and the Qwen2.5-0.5B `tokenizers` model on the same texts — outputs above. English ≈ 1.1 tokens/word on these samples; math/emoji inflate the count.
