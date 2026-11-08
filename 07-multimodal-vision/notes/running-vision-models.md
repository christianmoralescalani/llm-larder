# Running vision models — Ollama, llama.cpp, vLLM

**Summary.** All three runtimes accept images through the OpenAI-style chat API (base64 `image_url` content parts) plus native paths: Ollama CLI takes an image path argument, llama.cpp CLI takes `--image`, vLLM takes the same `image_url` parts. In llama.cpp the vision encoder lives in a separate **mmproj** file that must match the base model ([01-huggingface/notes/mmproj-vision.md](../../01-huggingface/notes/mmproj-vision.md)). Docker gotcha everywhere: image paths are resolved **inside the container** — mount the directory first.

## Ollama

CLI — pass the image path as an argument (Docker: mount the directory, path is inside the container):

```bash
docker run -d --name ollama -v ~/Pictures:/pictures ollama/ollama
docker exec -it ollama ollama run gemma3 /pictures/photo.jpg "describe this"
```

API — base64 in the `images` array:

```bash
IMG=$(base64 -w0 photo.jpg)
curl http://localhost:11434/api/chat -d "{
  \"model\": \"qwen2.5vl:7b\",
  \"messages\": [{\"role\": \"user\", \"content\": \"What chart type is this?\", \"images\": [\"$IMG\"]}]
}"
```

The OpenAI-compatible endpoint (`/v1/chat/completions`) also accepts standard `image_url` content parts. Full details: [03-ollama/notes/vision-models.md](../../03-ollama/notes/vision-models.md).

## llama.cpp

### CLI (llama-cli)

```bash
docker run --rm --gpus all -v ~/Downloads/models:/m -v ~/Pictures:/p \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  llama-cli -m /m/model.gguf --mmproj /m/mmproj-F16.gguf \
  --image /p/photo.jpg "what is in this image?"
```

- `--mmproj` is required (or use `-hf user/repo`, which auto-downloads model + mmproj).
- `--mmproj-device none` keeps the vision encoder on CPU (saves VRAM; slightly slower prefill) — useful when the LLM already fills the GPU.
- `--image` can be repeated (multi-image prompts).
- Token-budget dials: `--image-min-tokens` / `--image-max-tokens` (Qwen VL), `--mtmd-batch-max-tokens` (encoder batch → VRAM).

### Server (llama-server)

Serve with the same flags, then use the OpenAI API with a base64 `image_url`:

```bash
docker run -d --name llm -p 8080:8080 --gpus all \
  -v ~/Downloads/models:/models -v ~/Pictures:/pictures \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  --model /models/model.gguf --mmproj /models/mmproj-F16.gguf \
  --mmproj-device none --host 0.0.0.0 --port 8080 \
  --n-gpu-layers -1 --ctx-size 32768 --jinja

IMG=$(base64 -w0 ~/Pictures/photo.jpg)
curl http://localhost:8080/v1/chat/completions -d "{
  \"model\": \"qwen3.8\", \"stream\": false,
  \"messages\": [{\"role\": \"user\", \"content\": [
    {\"type\": \"text\", \"text\": \"Transcribe all the text in this image.\"},
    {\"type\": \"image_url\", \"image_url\": {\"url\": \"data:image/jpeg;base64,$IMG\"}}
  ]}]
}"
```

- The built-in web UI (http://localhost:8080) supports drag-and-drop images.
- **Video**: pass a video file the same way (or a frame sequence); `--video-fps N` controls frame sampling. Frames are just images — cost = frames × tokens/frame.
- This is the setup in `testcommands` at the repo root (Qwen3.8-27B + `mmproj-BF16.gguf` with `--mmproj-device none`).

## vLLM

```bash
# same image as text serving — vision models just work
docker run -d --name llm --gpus all -p 8000:8000 --ipc=host \
  vllm/vllm-openai:latest --model Qwen/Qwen2.5-VL-7B-Instruct
```

Send the same OpenAI `image_url` request as above (base64 data URL, or a public http URL the server can fetch). Tuning:

- `--limit-mm-per-prompt image=4` — cap images per request (protects your context).
- `--mm-processor-kwargs '{"max_pixels": 1003520}'` — cap the Qwen VL pixel budget (controls token cost).
- Vision models use more VRAM at load (encoder weights) and per image (KV) — see `notes/vision-vram.md`.

## Gotchas

1. **mmproj must match the base model** — mixing a 7B LLM with a 72B mmproj gives garbage output with no error message.
2. **Docker paths** — the image file must exist *inside* the container (mount the directory).
3. **Giant images** — a 4K photo is ~10,500 tokens on Qwen VL or 6,400 on Llama 3.2. Downscale first; cap with min/max pixels.
4. **Context math** — budget image tokens against `--ctx-size`; multi-image prompts eat context fast.
5. **mmproj precision** — F16/BF16 is the safe default; Q8_0 mmproj saves VRAM for little quality loss.
6. **Video is just frames** — a 10 s clip at 1 fps of 1080p frames ≈ 26k tokens on Qwen VL.

## Sources

- [llama.cpp — multimodal (mtmd) README](https://github.com/ggml-org/llama.cpp/blob/master/tools/mtmd/README.md)
- [llama.cpp server README (`--mmproj*`, `--image`, video)](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [Ollama — vision models (docs)](https://github.com/ollama/ollama/blob/main/docs/vision.md)
- [vLLM — multimodal inputs](https://docs.vllm.ai/en/stable/features/multimodal_inputs/)
- [This repo — 03-ollama/notes/vision-models.md](../../03-ollama/notes/vision-models.md)
- [This repo — 01-huggingface/notes/mmproj-vision.md](../../01-huggingface/notes/mmproj-vision.md)

## My experiments

- (fill in: e.g. "Qwen3.8-27B + mmproj-BF16 with `--mmproj-device none`: 1080p screenshot → TTFT 2.1 s (text-only 0.4 s), decode unchanged; moving the mmproj to GPU cut TTFT to 1.4 s; `--image-max-tokens 1024` halved the token cost and still read the table but dropped the footnote")
