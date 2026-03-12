# Image Generation & Editing

Generate and edit images via BlockRun's image API with x402 micropayments — no API keys, pay per image.

## Table of Contents

- [Quick Start](#quick-start)
- [Models & Pricing](#models--pricing)
- [API Reference](#api-reference)
  - [POST /v1/images/generations](#post-v1imagesgenerations)
  - [POST /v1/images/image2image](#post-v1imagesimage2image)
- [Code Examples](#code-examples)
  - [Image Generation](#image-generation-examples)
  - [Image Editing (img2img)](#image-editing-examples)
- [In-Chat Commands](#in-chat-commands)
- [Notes](#notes)

---

## Quick Start

ClawRouter runs a local proxy on port `8402` that handles x402 payments automatically. Point any OpenAI-compatible client at it:

```bash
curl -X POST http://localhost:8402/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana",
    "prompt": "a golden retriever surfing on a wave",
    "size": "1024x1024",
    "n": 1
  }'
```

Response:

```json
{
  "created": 1741460000,
  "data": [
    {
      "url": "https://files.catbox.moe/abc123.png"
    }
  ]
}
```

The returned URL is a publicly hosted image, ready to use in Telegram, Discord, or any client.

---

## Models & Pricing

| Model ID                   | Shorthand       | Price       | Max Size   | Provider          |
| -------------------------- | --------------- | ----------- | ---------- | ----------------- |
| `google/nano-banana`       | `nano-banana`   | $0.05/image | 1024×1024  | Google Gemini Flash |
| `google/nano-banana-pro`   | `banana-pro`    | $0.10/image | 4096×4096  | Google Gemini Pro |
| `openai/dall-e-3`          | `dall-e-3`      | $0.04/image | 1792×1024  | OpenAI DALL-E 3   |
| `openai/gpt-image-1`       | `gpt-image`     | $0.02/image | 1536×1024  | OpenAI GPT Image  |
| `black-forest/flux-1.1-pro`| `flux`          | $0.04/image | 1024×1024  | Black Forest Labs |

Default model: `google/nano-banana`.

---

## API Reference

### `POST /v1/images/generations`

OpenAI-compatible endpoint. Route via ClawRouter proxy (`http://localhost:8402`) for automatic x402 payment handling.

**Request body:**

| Field    | Type     | Required | Description                                      |
| -------- | -------- | -------- | ------------------------------------------------ |
| `model`  | `string` | Yes      | Model ID (see table above)                       |
| `prompt` | `string` | Yes      | Text description of the image to generate        |
| `size`   | `string` | No       | Image dimensions, e.g. `"1024x1024"` (default)  |
| `n`      | `number` | No       | Number of images (default: `1`)                  |

**Response:**

```typescript
{
  created: number;          // Unix timestamp
  data: Array<{
    url: string;            // Publicly hosted image URL
    revised_prompt?: string; // Model's rewritten prompt (dall-e-3 only)
  }>;
}
```

### `POST /v1/images/image2image`

Edit an existing image using AI. Route via ClawRouter proxy (`http://localhost:8402`) for automatic x402 payment handling.

**Request body:**

| Field   | Type     | Required | Description                                                    |
| ------- | -------- | -------- | -------------------------------------------------------------- |
| `model` | `string` | Yes      | Model ID (currently `openai/gpt-image-1`)                      |
| `prompt`| `string` | Yes      | Text description of the edit to apply                          |
| `image` | `string` | Yes      | Source image as base64 data URI (`data:image/png;base64,...`)   |
| `mask`  | `string` | No       | Mask image as base64 data URI (white = area to edit)           |
| `size`  | `string` | No       | Output dimensions, e.g. `"1024x1024"` (default)               |

**Response:**

```typescript
{
  created: number;          // Unix timestamp
  data: Array<{
    url: string;            // Locally cached image URL (http://localhost:8402/images/...)
    revised_prompt?: string; // Model's rewritten prompt
  }>;
}
```

---

## Code Examples

### Image Generation Examples {#image-generation-examples}

### curl

```bash
# Default model (nano-banana, $0.05)
curl -X POST http://localhost:8402/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google/nano-banana",
    "prompt": "a futuristic city at sunset, cyberpunk style",
    "size": "1024x1024",
    "n": 1
  }'

# DALL-E 3 with landscape size ($0.04)
curl -X POST http://localhost:8402/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/dall-e-3",
    "prompt": "a serene Japanese garden in autumn",
    "size": "1792x1024",
    "n": 1
  }'
```

### TypeScript / Node.js

```typescript
const response = await fetch("http://localhost:8402/v1/images/generations", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "google/nano-banana",
    prompt: "a golden retriever surfing on a wave",
    size: "1024x1024",
    n: 1,
  }),
});

const result = await response.json() as {
  created: number;
  data: Array<{ url: string; revised_prompt?: string }>;
};

const imageUrl = result.data[0].url;
console.log(imageUrl); // https://files.catbox.moe/xxx.png
```

### Python

```python
import requests

response = requests.post(
    "http://localhost:8402/v1/images/generations",
    json={
        "model": "google/nano-banana",
        "prompt": "a golden retriever surfing on a wave",
        "size": "1024x1024",
        "n": 1,
    }
)

result = response.json()
image_url = result["data"][0]["url"]
print(image_url)
```

### OpenAI SDK (drop-in)

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "blockrun",               // any non-empty string
  baseURL: "http://localhost:8402/v1",
});

const response = await client.images.generate({
  model: "google/nano-banana",
  prompt: "a golden retriever surfing on a wave",
  size: "1024x1024",
  n: 1,
});

console.log(response.data[0].url);
```

### startProxy (programmatic)

If you're using ClawRouter as a library:

```typescript
import { startProxy } from "@blockrun/clawrouter";

const proxy = await startProxy({ walletKey: process.env.BLOCKRUN_WALLET_KEY! });

const response = await fetch(`${proxy.baseUrl}/v1/images/generations`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "openai/dall-e-3",
    prompt: "a serene Japanese garden in autumn",
    size: "1792x1024",
    n: 1,
  }),
});

const { data } = await response.json();
console.log(data[0].url);

await proxy.close();
```

### Image Editing Examples {#image-editing-examples}

### curl

```bash
# Edit an image (gpt-image-1, $0.02)
curl -X POST http://localhost:8402/v1/images/image2image \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-image-1",
    "prompt": "add sunglasses to the person",
    "image": "data:image/png;base64,iVBOR...",
    "size": "1024x1024"
  }'

# With a mask (inpainting — white = area to edit)
curl -X POST http://localhost:8402/v1/images/image2image \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-image-1",
    "prompt": "replace the background with a sunset beach",
    "image": "data:image/png;base64,iVBOR...",
    "mask": "data:image/png;base64,iVBOR..."
  }'
```

### TypeScript / Node.js

```typescript
import { readFileSync } from "fs";

const imageBase64 = readFileSync("./photo.png").toString("base64");

const response = await fetch("http://localhost:8402/v1/images/image2image", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "openai/gpt-image-1",
    prompt: "change the background to a starry sky",
    image: `data:image/png;base64,${imageBase64}`,
    size: "1024x1024",
  }),
});

const result = (await response.json()) as {
  created: number;
  data: Array<{ url: string; revised_prompt?: string }>;
};

console.log(result.data[0].url); // http://localhost:8402/images/xxx.png
```

### Python

```python
import base64
import requests

with open("photo.png", "rb") as f:
    image_b64 = base64.b64encode(f.read()).decode()

response = requests.post(
    "http://localhost:8402/v1/images/image2image",
    json={
        "model": "openai/gpt-image-1",
        "prompt": "add a hat to the person",
        "image": f"data:image/png;base64,{image_b64}",
        "size": "1024x1024",
    },
)

result = response.json()
print(result["data"][0]["url"])
```

---

## In-Chat Commands

When using ClawRouter with OpenClaw, generate and edit images directly from any conversation:

### `/imagegen` — Generate images

```
/imagegen a dog dancing on the beach
/imagegen --model dall-e-3 a futuristic city at sunset
/imagegen --model banana-pro --size 2048x2048 mountain landscape
```

| Flag      | Default       | Description           |
| --------- | ------------- | --------------------- |
| `--model` | `nano-banana` | Model shorthand or ID |
| `--size`  | `1024x1024`   | Image dimensions      |

### `/img2img` — Edit images

```
/img2img --image ~/photo.png change the background to a starry sky
/img2img --image ./cat.jpg --mask ./mask.png remove the background
/img2img --image /tmp/portrait.png --size 1536x1024 add a hat
```

| Flag      | Default        | Description                           |
| --------- | -------------- | ------------------------------------- |
| `--image` | _(required)_   | Local image file path (supports `~/`) |
| `--mask`  | _(none)_       | Mask image (white = area to edit)     |
| `--model` | `gpt-image-1`  | Model to use                          |
| `--size`  | `1024x1024`    | Output size                           |

### Model shorthands

| Shorthand     | Full ID                     |
| ------------- | --------------------------- |
| `nano-banana` | `google/nano-banana`        |
| `banana-pro`  | `google/nano-banana-pro`    |
| `dall-e-3`    | `openai/dall-e-3`           |
| `gpt-image`   | `openai/gpt-image-1`       |
| `flux`        | `black-forest/flux-1.1-pro` |

---

## Notes

- **Local image caching** — All images (generated and edited) are cached locally at `~/.openclaw/blockrun/images/` and served via `http://localhost:8402/images/`. Both base64 data URIs and HTTP URLs from upstream are downloaded and replaced with localhost URLs.
- **Payment** — Each image costs the listed price in USDC, deducted from your wallet via x402. Make sure your wallet is funded before generating or editing.
- **No DALL-E content policy bypass** — DALL-E 3 and GPT Image 1 still apply OpenAI's content policy. Use `flux` or `nano-banana` for more flexibility with generation.
- **Size limits** — Requesting a size larger than the model's max will return an error. Check the table above before setting `--size`.
- **Image editing** — The `/v1/images/image2image` endpoint currently supports `openai/gpt-image-1`. The source image must be sent as a base64 data URI in the `image` field. The `/img2img` chat command handles file reading and base64 conversion automatically.
