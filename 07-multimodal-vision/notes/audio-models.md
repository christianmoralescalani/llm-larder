# Audio models — Whisper & friends (bonus)

**Summary.** Local audio splits into two camps: **ASR models** (Whisper, faster-whisper, whisper.cpp — speech → text; not LLMs, no sampling params, no chat template) and **audio-in LLMs** (Qwen2-Audio — audio becomes tokens, like vision, and the model answers in text). The practical local stack is Whisper → text → LLM: transcribe with faster-whisper or whisper.cpp, then feed the transcript into your local LLM for summarization/Q&A. TTS (speech *out* — Piper, Kokoro, XTTS) is a separate model family; here it only appears as the last pipeline stage.

## Whisper (the workhorse)

- OpenAI's ASR: encoder–decoder transformer, 99 languages, runs on CPU fine; GPU is 4–10× faster.
- Sizes: tiny (39M) → base (74M) → small (244M) → medium (769M) → **large-v3 (1.55B, ~3 GB VRAM — the quality pick)**.
- **faster-whisper** (CTranslate2): same models, ~4× faster, int8 on CPU — the default for local use.
- **whisper.cpp**: C++ port, `whisper-cli -m ggml-large-v3.bin -f audio.wav` — runs on anything, no Python.

```bash
# faster-whisper (Python) — the best local default
pip install faster-whisper
python3 -c "
from faster_whisper import WhisperModel
model = WhisperModel('large-v3', device='cuda', compute_type='float16')
segments, info = model.transcribe('meeting.mp3', vad_filter=True)
print(''.join(s.text for s in segments))
"

# whisper.cpp (no Python)
./whisper-cli -m ggml-large-v3.bin -f meeting.wav -l en
```

- **VAD filter** (`vad_filter=True` / `-f` auto-detects silence) skips silence — big speedup on long files.
- Output is **segments with timestamps** — keep them; that's what makes transcripts usable ("jump to 14:32").
- Language: `-l en` (or auto-detect) — forcing the language noticeably improves accuracy.

## Qwen2-Audio (audio-in LLM)

- 7B LLM that takes audio as input (audio → tokens, same idea as vision) and answers in text: transcription, audio QA ("what's the weather like in this recording?"), emotion/sound-event classification.
- Runs the same way as a vision model (llama.cpp with the audio mmproj, or HF/vLLM); GGUFs exist for Qwen2-Audio-7B.
- Pick this over Whisper when you need **understanding** (questions about the audio), not just a transcript. For pure transcription, Whisper is faster, cheaper, and more accurate.

## The local speech pipeline

```text
mic/recording → Whisper (ASR) → transcript → LLM (summarize/answer) → [TTS: Piper/Kokoro] → speaker
```

- **Latency**: Whisper large-v3 on a 4090-class GPU runs ~30× realtime; the LLM is usually the slower half of the pipeline.
- **Meeting-notes recipe**: record → faster-whisper with VAD → chunk the transcript → LLM with a structured prompt ("summarize + action items, JSON" — see [06 prompting](../06-sampling-and-prompting/notes/prompting-basics.md) for the schema-constraint part).
- **Full local speech-to-speech** (Whisper + LLM + TTS, all offline): see the video in [videos.md](../videos.md).

## Sources

- [Whisper (OpenAI, GitHub)](https://github.com/openai/whisper)
- [faster-whisper (CTranslate2)](https://github.com/SYSTRAN/faster-whisper)
- [whisper.cpp](https://github.com/ggml-org/whisper.cpp)
- [Qwen2-Audio](https://github.com/QwenLM/Qwen2-Audio)
- [This repo — 06 prompting basics (structured output for the LLM stage)](../06-sampling-and-prompting/notes/prompting-basics.md)

## My experiments

- (fill in: e.g. "1 h meeting on a 4090: faster-whisper large-v3 fp16 = 2 m 40 s with VAD (vs 9 m on CPU int8); transcript → Qwen3.8-27B with JSON schema → 0/20 parse failures; Qwen2-Audio-7B answered 'what decision was made?' correctly where the raw transcript alone needed 2 follow-up prompts")
