# Automatic Content Generate — Local LLM SPA

A privacy-first, **100% offline** single-page application that talks to a local
LLM through the [Ollama](https://ollama.com) local API. No prompt, no audio, no
data of any kind ever leaves your machine.

The frontend is intentionally a **single self-contained file** (`index.html`) —
HTML, CSS and Vanilla JS in one place — so it can be opened directly in a
browser or served by any static file server, with zero build step.

> Branding: **Automatic Content Generate**. This project contains no third-party
> assistant iframe, widget, logo, or branding. Everything rendered on screen is
> custom UI.

---

## Table of contents

1. [Architecture](#1-architecture)
2. [Hardware requirements](#2-hardware-requirements)
3. [Backend — install & run Ollama](#3-backend--install--run-ollama)
4. [Choose & download a model](#4-choose--download-a-model)
5. [Test the local API](#5-test-the-local-api)
6. [Privacy & CORS configuration](#6-privacy--cors-configuration)
7. [Run the frontend](#7-run-the-frontend)
8. [File layout](#8-file-layout)
9. [Extending to audio (TTS / STT)](#9-extending-to-audio-tts--stt)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Architecture

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│  Browser (index.html SPA)   │        │  Ollama local engine         │
│  Vanilla JS  ── fetch() ────┼───────▶│  http://localhost:11434      │
│  Dark + glassmorphism UI    │ ◀──────┤  /api/generate (stream JSON) │
└─────────────────────────────┘  NDJSON └──────────────────────────────┘
        100% on-device                         100% on-device
```

- The browser page talks **only** to `http://localhost:11434`.
- Ollama runs entirely on your machine; weights stay on disk, inference stays
  on CPU/GPU. Nothing is uploaded.
- Streaming uses Ollama's native newline-delimited JSON (`stream: true`), parsed
  chunk-by-chunk in the browser for a live typing effect.

---

## 2. Hardware requirements

Running an LLM locally is RAM/GPU bound. Pick the tier that matches your machine:

| Tier | RAM | GPU (VRAM) | Recommended models |
|------|-----|------------|--------------------|
| Minimum | 8 GB | none (CPU) | `phi3:mini` (3.8B), `qwen2.5:1.5b`, `tinyllama` (1.1B) |
| Recommended | 16 GB | 6 GB+ | `llama3.2:3b`, `mistral` (7B, q4) |
| Comfortable | 32 GB | 8–12 GB | `llama3.1:8b`, `mistral` (7B), `gemma2:9b` |

- CPU-only works, but expect ~3–10 tokens/sec for a 7B model.
- A discrete GPU gives 5–20× faster generation.
- Low-spec machine? Start with `qwen2.5:1.5b` or `phi3:mini` — both run
  comfortably under 4 GB RAM and still produce useful text.

---

## 3. Backend — install & run Ollama

### macOS

```bash
# Download the macOS app from https://ollama.com/download and drag to Applications,
# or use Homebrew:
brew install ollama

# Start the service (runs the API server on http://localhost:11434)
ollama serve
```

Opening the *Ollama.app* also starts `ollama serve` in the background.

### Windows

1. Download `OllamaSetup.exe` from <https://ollama.com/download> and run it.
2. The installer starts the service automatically (system tray icon).
3. Confirm with `ollama --version` in PowerShell.

### Linux

```bash
# One-line installer (official)
curl -fsSL https://ollama.com/install.sh | sh

# Start / enable the daemon
sudo systemctl enable --now ollama

# Or run it manually in the foreground
ollama serve
```

Verify the server is up:

```bash
curl http://localhost:11434/
# => Ollama is running
```

### GPU notes

- **macOS (Apple Silicon):** Ollama uses Metal automatically — no extra setup.
- **Linux:** install the matching NVIDIA driver + CUDA toolkit; Ollama detects
  `nvidia-smi` automatically. For AMD, the installer wires up ROCm.
- **Windows:** NVIDIA/AMD drivers are detected automatically by the desktop app.

---

## 4. Choose & download a model

Pull a model (one-time download; the file is cached in `~/.ollama/models`):

```bash
# Lightweight — great for low-RAM machines
ollama pull qwen2.5:1.5b      # ~1 GB
ollama pull phi3:mini         # ~2.3 GB

# Balanced
ollama pull llama3.2:3b       # ~2 GB
ollama pull mistral           # ~4.1 GB (7B, q4)

# Larger / higher quality
ollama pull llama3.1:8b       # ~4.7 GB
```

List installed models:

```bash
ollama list
```

Run a quick interactive chat to confirm the model works:

```bash
ollama run mistral "Tulis satu kalimat tentang kopi pagi hari."
```

The frontend lets you switch models from the UI sidebar, but you must `ollama pull`
each model you want to use **before** selecting it in the app.

---

## 5. Test the local API

Ollama exposes a simple REST API. The endpoint the frontend uses is
`POST http://localhost:11434/api/generate`.

### One-shot (no streaming)

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "mistral",
  "prompt": "Jelaskan apa itu glassmorphism dalam 2 kalimat.",
  "stream": false
}'
```

Response (truncated):

```json
{
  "model": "mistral",
  "response": "Glassmorphism adalah gaya desain ...",
  "done": true,
  "total_duration": 1840000000
}
```

### Streaming (what the SPA uses)

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "mistral",
  "prompt": "Hitung sampai 5.",
  "stream": true
}'
```

Ollama returns **newline-delimited JSON** — one JSON object per token chunk,
each ending with `"done": false` until the final `"done": true` object.

There is also a chat-style endpoint you can use instead:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "mistral",
  "messages": [{"role":"user","content":"Halo!"}],
  "stream": true
}'
```

---

## 6. Privacy & CORS configuration

### Why nothing leaves your machine

- Model weights are stored locally under `~/.ollama/models` (or
  `%USERPROFILE%\.ollama\models` on Windows). Inference is on-device.
- The Ollama API listens on `127.0.0.1` by default — only your own machine can
  reach it.
- The SPA only calls `http://localhost:11434`. There is no telemetry, analytics,
  CDN, or external font in `index.html`. Open the page once and it works offline.

### CORS — letting the browser talk to localhost

Browsers block cross-origin requests by default. Ollama allows origins via the
`OLLAMA_ORIGINS` environment variable. For local development allow `null` (file://)
and the common static-server origins:

```bash
# macOS / Linux — restart ollama serve with the env var
OLLAMA_ORIGINS="*" ollama serve
```

For a permanent setting on Linux (systemd):

```bash
sudo systemctl edit ollama
# add:
# [Service]
# Environment="OLLAMA_ORIGINS=*"
sudo systemctl restart ollama
```

> On macOS, the desktop app reads `OLLAMA_ORIGINS` if you launch it from a shell:
> ```bash
> launchctl setenv OLLAMA_ORIGINS "*"
> ```
> then quit & reopen Ollama.app.

For a **strictly local** deployment you can restrict origins instead of using `*`:

```bash
OLLAMA_ORIGINS="http://localhost:8000,http://127.0.0.1:8000,null" ollama serve
```

### Binding / network exposure

Ollama binds to `127.0.0.1` by default, so the API is **not** reachable from other
machines. Do **not** set `OLLAMA_HOST=0.0.0.0` unless you intend to expose it on
your LAN and have secured it — that would break the "100% offline / local" goal.

---

## 7. Run the frontend

You have two options.

### Option A — open the file directly

Double-click `index.html`, or:

```bash
# macOS
open index.html
# Linux
xdg-open index.html
# Windows
start index.html
```

> When opening via `file://`, the page origin is `null`. That is why the CORS
> setup above includes `null` in `OLLAMA_ORIGINS`. Using a local server
> (Option B) is recommended and avoids any origin quirks.

### Option B — serve with a tiny static server (recommended)

```bash
# Python 3 (no install needed)
python3 -m http.server 8000

# or Node
npx serve .
```

Then visit **http://localhost:8000**.

Using the app:

1. The header shows the branding **Automatic Content Generate**.
2. The sidebar lists common models; type any model name you've pulled.
3. The status pill in the header shows whether Ollama is reachable.
4. Type a prompt, press Enter (or click **Generate**). The response streams in
   token-by-token.
5. Conversation history is kept in-memory only — nothing is persisted, nothing
   is sent anywhere. Reload to wipe it.

---

## 8. File layout

```
.
├── index.html      # The entire SPA (HTML + CSS + Vanilla JS)
└── README.md       # This guide (backend install + privacy + run steps)
```

That's the whole project — one app file plus this guide.

---

## 9. Extending to audio (TTS / STT)

The architecture is deliberately a single-page app so you can bolt on local
audio modules later without restructuring anything.

- **Text → Speech (TTS):** run [Piper](https://github.com/rhasspy/piper) or
  [Coqui XTTS](https://github.com/coqui-ai/TTS) locally. Pipe the streamed text
  (or final response) into the TTS engine and play the resulting WAV via an
  `<audio>` element. Everything stays on-device.
- **Speech → Text (STT):** use [whisper.cpp](https://github.com/ggerganov/whisper.cpp)
  (GGML, CPU/GPU) with a small model (`tiny` or `base`) for a local "dictate
  prompt" button. Capture audio with the `MediaRecorder` API, send the file to
  your local STT server, and feed the transcript into the existing prompt box.

Both add-ons can be wired into the same `index.html` without changing the
Ollama integration.

---

## 10. Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Failed to fetch` in the browser | Ollama not running, or CORS blocking. Run `OLLAMA_ORIGINS="*" ollama serve` and reload. |
| Status pill shows **offline** | Confirm `curl http://localhost:11434/` returns `Ollama is running`. |
| `model not found` | You forgot to `ollama pull <model>`. Run `ollama list` to see what's installed. |
| Generation is very slow | Use a smaller model (`qwen2.5:1.5b`) or enable GPU (see §3 GPU notes). |
| Page opened via `file://` can't connect | Use a local server (Option B) or add `null` to `OLLAMA_ORIGINS`. |
| Port 11434 already in use | Another Ollama instance is running. `ollama serve` only needs one. |

---

Built to run entirely on your machine. Your prompts, your audio, your data —
they never leave the device.
