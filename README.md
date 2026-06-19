# tangerine

A local web chat interface over [Ollama](https://ollama.com), designed to feel like a messaging app rather than an AI tool. Runs on-device, accessible to any device on the same LAN.

---

## Stack

| Layer | Tech |
|---|---|
| Server | Python 3 · FastAPI · httpx · uvicorn |
| Frontend | Single HTML file — no build step, no CDN |
| LLM runtime | Ollama on `localhost:11434` |

---

## Features

- **Streaming** — tokens appear as they're generated via SSE (`fetch` + `ReadableStream`)
- **Multi-turn memory** — full conversation history sent with every request
- **Paragraph batching** — responses split at `\n\n` boundaries; each paragraph arrives as its own bubble
- **Think toggle** — ⚡ fast mode skips model reasoning for low latency; click to enable extended thinking for complex queries
- **Context trimming** — oldest messages dropped when history exceeds `MAX_CONTEXT_CHARS`; frontend notified via a `trimmed` event
- **Persistence** — conversation stored in `localStorage`, survives page refresh
- **LAN access** — server binds to `0.0.0.0:8080`

---

## Setup

Requires Ollama running locally with `gemma4:e4b` pulled.

```bash
pip install -r requirements.txt
python3 server.py
```

| Access | URL |
|---|---|
| Local | `http://localhost:8080` |
| LAN | `http://<host-ip>:8080` |

---

## Project Structure

```
tangerine/
├── server.py        # FastAPI app — SSE proxy, context trimming
├── static/
│   └── index.html   # Entire frontend — HTML + CSS + JS
└── requirements.txt
```

---

## Configuration

All constants are at the top of `server.py`.

| Constant | Default | Description |
|---|---|---|
| `OLLAMA_BASE` | `http://localhost:11434` | Ollama endpoint |
| `MODEL` | `gemma4:e4b` | Hardcoded model — no selector by design |
| `MAX_CONTEXT_CHARS` | `400_000` | Context budget (~100k tokens); trims oldest messages when exceeded |

Server port is set in the `uvicorn.run()` call at the bottom of `server.py`.

---

## Architecture

```
Browser
  │
  └─ POST /api/chat  { messages, think }
        │
        ├─ trim_to_context()          # drop oldest msgs if over budget
        │
        └─ httpx stream → Ollama /api/chat
                │
                └─ SSE events ──────► Browser (fetch + ReadableStream)
                     content | done | trimmed
```

The server is stateless — conversation history lives entirely in the browser (`localStorage`). Every request carries the full history; the server trims it to fit the model's context budget before forwarding to Ollama.

Thinking tokens emitted by the model (`message.thinking`) are consumed and discarded server-side; only `message.content` is relayed to the client.

---

## SSE Event Schema

`POST /api/chat` returns `text/event-stream`. Each event is a JSON-encoded line:

```
data: {"type": "content", "text": "..."}   # token chunk
data: {"type": "done"}                      # generation complete
data: {"type": "trimmed"}                   # context was trimmed this turn
```

---

## Frontend Notes

- No framework, no bundler — all UI is in `static/index.html`
- Markdown rendered post-stream (after `done` event), not mid-stream
- Paragraph batching tracks code fence depth to avoid splitting inside ` ``` ` blocks
- Think mode state persisted in `localStorage` key `tangerine_think`
- `font-size: 16px` on mobile inputs prevents iOS auto-zoom
- `100dvh` used instead of `100vh` for correct mobile keyboard handling
- `env(safe-area-inset-*)` applied to header and footer for iPhone notch/home bar
