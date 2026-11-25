---
title: Qwen2API
emoji: 🚀
colorFrom: blue
colorTo: indigo
sdk: docker
pinned: false
---

<h1 align="center">Qwen2API</h1>

<p align="center">
  <em>A proxy that exposes Qwen Chat through an OpenAI-compatible API.</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Node.js-%E2%89%A518-339933?logo=nodedotjs&logoColor=white" alt="Node.js >= 18" />
  <img src="https://img.shields.io/badge/Express-5-000000?logo=express&logoColor=white" alt="Express 5" />
  <img src="https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white" alt="Dockerfile included" />
  <img src="https://img.shields.io/badge/Cloudflare_Workers-supported-F38020?logo=cloudflare&logoColor=white" alt="Cloudflare Workers" />
</p>

<p align="center">
  English · <a href="README_ZH.md">中文文档</a>
</p>

> [!IMPORTANT]
> **Attribution.** This repository is a copy of [**smanx/qwen2api**](https://github.com/smanx/qwen2api), created and maintained by [@smanx](https://github.com/smanx). All credit for the code goes to the original author. For the latest version, issues, and the author's public test instance, please use the upstream repository.

---

## Overview

Qwen2API translates OpenAI-style requests (`/v1/models`, `/v1/chat/completions`, `/v1/images/generations`) into calls to Qwen Chat, so existing OpenAI SDKs and tools can talk to Qwen models by changing only the base URL. The same core logic (`core.js`) runs on a local/Docker Express server, Vercel, Netlify Edge Functions, and Cloudflare Workers.

## Features

- **OpenAI-compatible endpoints** for models, chat completions, and image generation
- **Streaming** responses over Server-Sent Events
- **Optional API-key auth** via `API_TOKENS` (comma-separated)
- **Multimodal input**: images, files, and audio as OpenAI-style content parts (uploaded through Qwen's own upload flow)
- **Image generation** with ratio (`16:9`) or OpenAI-style (`1024x1024`) sizes, `url` or `b64_json` output
- **Video URL analysis** via `yt-dlp` (Docker / local server only)
- **Built-in web chat UI** at `/chat` with streaming, attachments, a log panel, and a ZH/EN toggle
- **Userscript** (`scripts/tampermonkey.js`) that adds a video-analysis panel to Bilibili and YouTube pages
- **Multi-platform deployment**: Docker, Hugging Face Spaces, Vercel, Netlify, Cloudflare Workers

> [!NOTE]
> Tool / function calling is **not** implemented.

## Tech Stack

| Area | Technology |
|---|---|
| Runtime | Node.js ≥ 18 (Docker image uses `node:24-alpine`) |
| Server | Express 5 |
| Serverless | Vercel (`api/index.js`), Netlify Edge Functions (`netlify/edge-functions/api.js`), Cloudflare Workers (`worker.js`) |
| Media | `ffmpeg` + `yt-dlp` (installed in the Docker image) |

## Getting Started

### Local development

```bash
npm install
npm run dev     # nodemon, detailed logs on, http://localhost:8765
# or
npm start       # node index.js
```

### Docker

The image listens on port `7860` (set in the `Dockerfile`).

```bash
docker build -t qwen2api .
docker run -d -p 7860:7860 -e API_TOKENS=your_token qwen2api
```

### Hugging Face Spaces

1. Create a new **Docker** Space.
2. Push this repository to the Space (the YAML header at the top of this file configures it).
3. Optionally set `API_TOKENS` in the Space's variables/secrets.

### Vercel / Netlify

Import the repository in Vercel or Netlify; the routing is already configured in `vercel.json` and `netlify.toml`. Optionally set `API_TOKENS`.

### Cloudflare Workers

```bash
npm install -g wrangler
wrangler login
wrangler deploy        # or: npm run deploy-worker
```

Set `API_TOKENS` in the Cloudflare dashboard.

> [!WARNING]
> Video URL analysis and very large files are not supported on serverless deployments (Vercel, Netlify, Cloudflare Workers). Use Docker or the local server for those.

## Configuration

| Variable | Description | Default |
|---|---|---|
| `API_TOKENS` | Allowed API keys, comma-separated. Empty = no auth. | empty |
| `PORT` | Express server port | `8765` (`7860` in Docker) |
| `CHAT_DETAIL_LOG` | Verbose chat/upload logs (`true`/`1`/`on`/`yes`) | off |
| `JSON_BODY_LIMIT` | Express JSON body limit (local/Docker only) | `20mb` |
| `MIN_VIDEO_RESOLUTION` | Minimum resolution used for video analysis | `480` |
| `YT_DLP_PATH` | Path to a `yt-dlp` binary | auto-detect |
| `YT_DLP_AUTO_DOWNLOAD` | Download `yt-dlp` automatically if missing | `true` |

## API

| Endpoint | Method | Description |
|---|---|---|
| `/v1/models` | GET | List models |
| `/v1/chat/completions` | POST | Chat completion (streaming supported) |
| `/v1/chat/completions/log` | POST | Chat completion with streamed server logs (used by the web UI) |
| `/v1/images/generations` | POST | Image generation |
| `/chat` | GET | Built-in web chat UI |
| `/` | GET | Health check |

### Example

```bash
curl https://your-domain/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_token" \
  -d '{
    "model": "qwen3.5-plus",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

With the OpenAI Python SDK:

```python
from openai import OpenAI

client = OpenAI(api_key="your_token", base_url="https://your-domain/v1")
stream = client.chat.completions.create(
    model="qwen3.5-plus",
    messages=[{"role": "user", "content": "Hello!"}],
    stream=True,
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="")
```

### Multimodal content parts

`messages[].content` accepts arrays with parts such as:

- `{"type":"text","text":"..."}` / `{"type":"input_text","input_text":"..."}`
- `{"type":"image_url","image_url":{"url":"https://..."}}` / `{"type":"input_image","image_url":"https://..."}`
- `{"type":"file","file_data":"data:...base64,...","filename":"a.pdf"}` / `{"type":"input_file", ...}`
- `{"type":"audio","file_data":"https://..."}` / `{"type":"input_audio", ...}`

Legacy message-level `files` / `attachments` arrays are also accepted.

### Image generation

| Parameter | Type | Required | Notes |
|---|---|---|---|
| `prompt` | string | yes | Image description |
| `model` | string | no | Default `qwen3.5-plus` |
| `n` | number | no | 1–10, default 1 |
| `size` | string | no | `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, or `WIDTHxHEIGHT` (mapped to the closest ratio). Default `1:1` |
| `response_format` | string | no | `url` (default) or `b64_json` |

## Project Structure

```
.
├── core.js                       # Core request translation logic
├── index.js                      # Express server (local / Docker / HF Spaces)
├── api/index.js                  # Vercel entry
├── netlify/edge-functions/api.js # Netlify Edge Function entry
├── worker.js                     # Cloudflare Workers entry
├── chat.html                     # Built-in web chat UI
├── scripts/tampermonkey.js       # Video-analysis userscript
├── video-analyzer.user.js        # Userscript variant
├── tasks/                        # Development notes (Chinese)
├── Dockerfile · vercel.json · netlify.toml · wrangler.toml
└── package.json
```

## Disclaimer

For learning and testing only. Not intended for production or commercial use; users are responsible for how they use it and for complying with Qwen's terms of service.

## License

No license file is included in this repository or in the upstream project. Refer to [smanx/qwen2api](https://github.com/smanx/qwen2api) for the author's terms.

## Credits

- Original project: [smanx/qwen2api](https://github.com/smanx/qwen2api) by [@smanx](https://github.com/smanx)
- Repository maintained by [@aghorbani84](https://github.com/aghorbani84)
