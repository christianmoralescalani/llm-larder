# MTP (Multi-Token Prediction)

**Summary.** Multi-token prediction is a training technique where the model is trained to predict not just the *next* token but the next *k* tokens in parallel (separate small "MTP modules" predict token t+2, t+3, …). It improves training efficiency and model quality, and at inference time the extra modules can optionally be used to *speculatively* verify multiple tokens in one step — a free-ish speedup. DeepSeek-V3/R1 ships with MTP depth 1 (one extra module predicting one token ahead).

## Next-token vs multi-token prediction

Standard training: at every position, predict only the next token.

MTP training: at every position, also predict the next 2nd, 3rd, … token:

```text
input:  [t1, t2, ..., tn]
main LM head → predicts t_{n+1}
MTP module 1 → predicts t_{n+2}  (conditioned on t_{n+1})
MTP module 2 → predicts t_{n+3}  (conditioned on t_{n+1}, t_{n+2})
...
```

Each MTP module is small (a small attention block + projection + shared LM head). In DeepSeek-V3 there is **one** MTP module (depth 1): it predicts one extra token ahead, using the hidden state plus the predicted next token.

## Why it helps

1. **Training efficiency**: the auxiliary prediction targets give the model denser gradient signal — the model has to form better internal representations to predict further ahead. DeepSeek's ablations show MTP improves the main objective and final quality (DeepSeek-V3 report §4.5.1).
2. **Inference speedup (speculative use)**: at decode time, the MTP module proposes the next token while the main model verifies it. If correct, you got 2 tokens out of 1 forward pass. This is a form of *self-speculative decoding* — no separate draft model needed. DeepSeek reports ~1.8× decode speedup from the MTP module in their serving stack (used with their own engine).
3. **Quality**: models trained with MTP tend to generalize slightly better, per the DeepSeek ablations.

## What you'll see in practice

- **Model names/configs**: DeepSeek-V3/R1 `config.json` shows `num_nextn_predict_layers: 1`. GGUF builds of R1 often include or exclude the MTP layer depending on the converter; llama.cpp's support for MTP-assisted decoding has been evolving — check your build.
- **Speed**: the speedup only materializes if your runtime actually uses the MTP module for speculation. If it's just present in the file, it costs a little extra memory and nothing else.
- **Not to be confused with**: "speculative decoding" with a separate small draft model (different technique, same idea — propose several tokens, verify in parallel). MTP is the model *being its own draft model*.

## Why it matters for local inference

- Mostly a "nice to know": it explains why R1/V3 files have an extra module and why serving speed claims beat naive math.
- If you're running DeepSeek R1 locally and see "MTP" in logs or flags, that's this.
- The general lesson: **verification is parallel, generation is sequential** — any technique that proposes several candidate tokens at once and checks them in one pass is a real speedup. That's the core idea behind speculative decoding (topic 04).

## Sources

- DeepSeek-V3 Technical Report, §2.1.3 (Multi-Token Prediction) and §4.5.1 (ablations) — https://arxiv.org/abs/2412.19437
- DeepSeek R1 paper (same architecture, MTP depth 1) — https://arxiv.org/abs/2501.12948

## My experiments

- (fill in: e.g. "checked whether my llama.cpp build uses the R1 MTP layer and what the speed delta is")
