# Vision models in Ollama

**Summary.** Ollama's vision models accept images alongside text. CLI: pass a file path as an argument — in Docker the path is resolved **inside the container**, so mount the directory first. REST: base64-encoded images in the `images` array. The workhorses: `llava`, `gemma3` (multimodal by default), `qwen2.5vl`/`qwen3-vl`, `llama3.2-vision`.

## Which models

| Model | Sizes | Notes |
|---|---|---|
| `llava` | 7B (default), 13B | classic; `llava:13b` is noticeably better |
| `gemma3` | 4B, 12B, 27B | multimodal by default — no separate tag needed |
| `qwen2.5vl` / `qwen3-vl` | 7B–72B | strong OCR / document reading |
| `llama3.2-vision` | 11B, 90B | Meta's vision line |

Browse: [ollama.com/library](https://ollama.com/library) (filter: multimodal).

## CLI (via docker exec)

```bash
# the image path is resolved INSIDE the container — mount the directory first:
docker run … -v ~/Pictures:/pictures …

docker exec -it ollama ollama run gemma3 /pictures/smile.png "what is in this image?"
docker exec -it ollama ollama run llava:13b /pictures/chart.png "summarize this chart"
```

If the file is only on the host: `docker cp ~/img.png ollama:/tmp/img.png` first.

## REST API (base64)

```bash
IMG=$(base64 < ~/Pictures/smile.png | tr -d '\n')

curl http://localhost:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "gemma3",
  "messages": [{
    "role": "user",
    "content": "What is in this image?",
    "images": ["'"$IMG"'"]
  }],
  "stream": false
}'
```

- REST expects base64 strings; the Python/JS SDKs accept file paths, URLs, or raw bytes.
- Multiple images: add more entries to `images`.
- `OLLAMA_MAX_BODY_SIZE` (default 100M) caps the request — big photos are fine, but a 4K burst will hit it.

## Python (from the host)

```python
from ollama import chat
import base64

img = base64.b64encode(open("smile.png", "rb").read()).decode()

r = chat(
    model="gemma3",
    messages=[{"role": "user",
               "content": "What is in this image? Be concise.",
               "images": [img]}],
)
print(r.message.content)
```

Point the client at the container: `client = Client(host="http://localhost:11434")` (from the host) or `http://ollama:11434` (sibling container).

## VRAM

Vision models need more headroom than text-only: the image is encoded into visual tokens that eat context (an image ≈ hundreds to thousands of "tokens" depending on resolution). Budget accordingly — see [07-multimodal-vision](../../07-multimodal-vision/) for the math and the [02-hardware-and-vram budget table](../../02-hardware-and-vram/cheatsheets/vram-budget.md).

## Sources

- [Ollama vision guide](https://docs.ollama.com/capabilities/vision)
- [API — chat (images)](https://docs.ollama.com/api/chat)
- [Ollama library](https://ollama.com/library)

## My experiments

- (fill in: e.g. "gemma3:12b on a 24 GB card: photo description in ~3 s after load; 4K screenshot hit the 100M body limit → downscale first")
