# 🛍️ Daraz Voice Assistant

**CS 4063 — Natural Language Processing | Assignment 3**
Due: March 29, 2026

> A low-latency, voice-to-voice conversational shopping assistant running entirely on CPU using locally deployed open-weights models. No RAG, no external APIs, no cloud inference.

**Live Demo:** `https://your-vercel-link-here.vercel.app` ← Rayan: update this after deploying

---

## 👥 Team

| Member | Role | Files Owned |
|--------|------|-------------|
| **Awwab** | Backend & Infrastructure | `main.py`, `routes.py`, `config.py`, `context.py`, `database.py`, `Dockerfile`, `docker-compose.yml`, `locustfile.py` |
| **Uwaid** | Model Engine | `engine.py` (STT + LLM + TTS implementations), `models/` |
| **Rayan** | Frontend | `frontend/` (React voice UI, MediaRecorder, audio playback) |

---

## 📋 Assignment Constraints Met

| Constraint | How |
|-----------|-----|
| No RAG | Pure prompt engineering + sliding window context |
| No external tools | All models run locally via llama-cpp-python and faster-whisper |
| Local CPU deployment | `n_gpu_layers=0`, quantized GGUF (Q4_K_M), int8 Whisper |
| Real-time streaming < 1s | WebSocket binary streaming, ThreadPoolExecutor for non-blocking inference |
| Up to 4 concurrent users | Async FastAPI + single ThreadPool worker + Redis session isolation |
| Conversation state | STATE tag extraction → Redis hot cache → SQLite cold storage |
| Clean API | FastAPI with Swagger at `/docs`, full Postman collection included |
| ChatGPT-style UI | Rayan's React frontend with voice input and audio playback |

---

## 🏗️ Architecture

```
Browser (Rayan's React UI)
    │
    │  WebSocket  ws://host/ws/chat?session_id=uuid
    │  binary frames (WebM/WAV audio) ──► backend
    │  binary frames (WAV audio)      ◄── backend
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  nginx  (port 80)                                   │
│  • Reverse proxy to backend                         │
│  • Serves frontend static files                     │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│  FastAPI Backend  (port 8000)          [Awwab]       │
│                                                     │
│  Middleware stack (request order):                  │
│    1. PayloadSizeLimitMiddleware  → reject > 1 MB   │
│    2. SlowAPIMiddleware           → per-IP limits   │
│    3. CORSMiddleware              → strict origin   │
│                                                     │
│  /ws/chat  ──► VoiceEngine.process_audio()          │
│                      │                              │
│            ┌─────────▼──────────┐                  │
│            │  transcribe_audio  │  [Uwaid]          │
│            │  faster-whisper    │                   │
│            └─────────┬──────────┘                  │
│            ┌─────────▼──────────┐                  │
│            │  _generate_text    │  [Awwab orch /    │
│            │  llama-cpp Qwen    │   Uwaid model]    │
│            └─────────┬──────────┘                  │
│            ┌─────────▼──────────┐                  │
│            │  synthesize_speech │  [Uwaid]          │
│            │  Kokoro / Piper    │                   │
│            └─────────┬──────────┘                  │
│                      │ WAV bytes                    │
└──────────────────────┼──────────────────────────────┘
                       │
        ┌──────────────▼──────────────┐
        │  Redis  (session hot cache) │  [Awwab]
        │  SQLite (cold persistence)  │
        └─────────────────────────────┘
```

---

## 📁 Project Structure

```
project-root/
│
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   └── routes.py        # WebSocket, REST endpoints, JWT stubs, rate limits
│   │   ├── core/
│   │   │   └── config.py        # All constants, security config, token guardrail
│   │   ├── llm/
│   │   │   └── engine.py        # ⚠️  Uwaid: implement STT/TTS here — stubs with TODOs
│   │   ├── memory/
│   │   │   ├── context.py       # Redis session management (replaces A2 dict)
│   │   │   └── database.py      # SQLite persistence + bleach sanitization
│   │   └── main.py              # FastAPI app, all middleware, lifespan
│   ├── Dockerfile
│   └── requirements.txt
│
├── frontend/                    # ⚠️  Rayan: React voice UI goes here
│   └── index.html
│
├── models/                      # ⚠️  Uwaid: place .gguf file here (git-ignored)
│   └── .gitkeep
│
├── nginx/
│   └── nginx.conf
│
├── data/                        # Auto-created — sessions.db lives here (git-ignored)
│
├── results/                     # Locust CSV output goes here
│
├── locustfile.py                # Load testing — 4 concurrent users
├── daraz_assistant_a3.postman_collection.json
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 🚀 Quick Start

### Prerequisites

- Docker Desktop (includes Docker Compose)
- The GGUF model file in `./models/` — Uwaid's responsibility
- Git

### 1. Clone

```bash
git clone <repo-url>
cd <repo>
```

### 2. Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

```env
FRONTEND_ORIGIN=http://localhost:3000
JWT_SECRET=replace_with_any_long_random_string
REDIS_URL=redis://redis:6379/0
MODEL_PATH=/models/qwen2.5-3b-instruct-q4_k_m.gguf
```

### 3. Start everything

```bash
docker compose up --build
```

Services start in dependency order: Redis → Backend → nginx.
Watch for `✅ Startup complete. API ready.` in backend logs before testing.

| Service | URL |
|---------|-----|
| Swagger UI | http://localhost:8000/docs |
| Frontend | http://localhost/ui |
| Health check | http://localhost:8000/health |

### 4. Verify

```bash
curl http://localhost:8000/health
# {"status":"ok","redis":"ok","active_sessions":0}
```

---

## 🔧 Local Dev (no Docker)

```bash
cd backend
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi "uvicorn[standard]" uvloop
pip install llama-cpp-python    # compiles C++ — takes 2-5 min
pip install redis slowapi bleach "python-jose[cryptography]"
pip install locust websocket-client

# Freeze
pip freeze > requirements.txt

# Run Redis in Docker while developing locally
docker run -d -p 6379:6379 redis:7.2-alpine

# Start backend
uvicorn app.main:app --reload --port 8000
```

---

## 📡 API Reference

### WebSocket — `ws://host/ws/chat`

The primary voice endpoint. Connect with your session ID in the query string:

```
ws://localhost:8000/ws/chat?session_id=<uuid>
```

**Voice mode** (binary frames — primary A3 flow):

```
Client ──► binary frame   raw audio bytes (WebM/WAV from MediaRecorder)
Client ◄── binary frame   WAV audio bytes (TTS response)
Client ◄── text frame     {"event":"turn_complete","status":"active","turns_used":1,"turns_max":15}
```

**Text fallback mode** (JSON frames — Postman / debug):

```
Client ──► text frame  {"session_id":"uuid","message":"I need a laptop"}
Client ◄── text frame  {"token":"Great","done":false}    ← one per token
Client ◄── text frame  {"token":"","done":true,"full_response":"...","latency_ms":1240}
```

### REST Endpoints

| Method | Path | Rate Limit | Description |
|--------|------|-----------|-------------|
| `GET` | `/health` | — | Server + Redis status |
| `POST` | `/chat` | 10/min | Text-in/text-out fallback |
| `GET` | `/session/welcome/{id}` | 30/min | Welcome message + session init |
| `POST` | `/reset` | 20/min | Clear session (New Chat button) |
| `GET` | `/sessions` | 30/min | List all sessions from SQLite |
| `GET` | `/sessions/{id}` | 30/min | Full history + state |
| `DELETE` | `/sessions/{id}` | 20/min | Delete permanently |
| `POST` | `/auth/login` | 5/min | JWT → HttpOnly cookie |
| `POST` | `/auth/refresh` | 10/min | Refresh JWT |
| `POST` | `/auth/logout` | — | Clear auth cookie |
| `POST` | `/benchmark?runs=N` | 5/min | Latency benchmark (N = 1–20) |
| `POST` | `/debug/warmup` | — | Warm up LLM KV cache |
| `GET` | `/debug/state/{id}` | — | Extracted budget/item/preferences |
| `GET` | `/debug/history/{id}` | — | Raw message history |

---

## 🔒 Security

| Feature | File | What it does |
|---------|------|-------------|
| Payload size limit | `main.py` | Rejects bodies > 1 MB with HTTP 413 before any handler runs |
| Rate limiting | `main.py` + `routes.py` | Per-IP limits via `slowapi` — 10/min chat, 5/min login |
| Strict CORS | `main.py` | Only `FRONTEND_ORIGIN` env var is allowed, no wildcard |
| Input sanitization | `routes.py` + `database.py` | `bleach.clean()` strips all HTML/JS before LLM and before DB write |
| Token truncation | `config.py` | Hard-caps user input to `N_CTX / 2` tokens — prevents context overflow attacks |
| JWT + HttpOnly cookies | `routes.py` | Tokens in HttpOnly cookies — inaccessible to JavaScript (XSS-safe) |
| Redis session isolation | `context.py` | Namespaced keys per session — concurrent users fully isolated |
| Non-root Docker user | `Dockerfile` | App runs as `appuser`, not root |

---

## 🤖 Uwaid — What You Need To Implement

Everything is in **`backend/app/llm/engine.py`**. Search `# TODO Uwaid:` — there are exactly 3 stubs.

### Task 1 — Load the LLM

Find `_llm = None` at the top of `engine.py` and replace it:

```python
from llama_cpp import Llama
from app.core.config import N_CTX, N_THREADS, N_BATCH
import os

_llm = Llama(
    model_path=os.getenv("MODEL_PATH", "models/qwen2.5-3b-instruct-q4_k_m.gguf"),
    n_ctx=N_CTX,
    n_threads=N_THREADS,
    n_batch=N_BATCH,
    n_gpu_layers=0,
    verbose=False,
)
```

### Task 2 — Implement `transcribe_audio()` (Whisper STT)

```python
from faster_whisper import WhisperModel
import tempfile, asyncio

_stt_model = WhisperModel("small", device="cpu", compute_type="int8")

async def transcribe_audio(audio_bytes: bytes, session_id: str) -> str:
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        f.write(audio_bytes)
        tmp_path = f.name
    loop = asyncio.get_event_loop()
    segments, _ = await loop.run_in_executor(
        _executor, lambda: _stt_model.transcribe(tmp_path, language="en")
    )
    return " ".join(s.text for s in segments).strip()
```

Install: `pip install faster-whisper`

### Task 3 — Implement `synthesize_speech()` (TTS)

```python
from kokoro_onnx import Kokoro
import soundfile as sf, io, asyncio

_tts_model = Kokoro("kokoro-v0_19.onnx", "voices.bin")

async def synthesize_speech(text: str, session_id: str) -> bytes:
    loop = asyncio.get_event_loop()
    samples, sample_rate = await loop.run_in_executor(
        _executor, lambda: _tts_model.create(text, voice="af_heart", speed=1.0)
    )
    buf = io.BytesIO()
    sf.write(buf, samples, sample_rate, format="WAV")
    return buf.getvalue()
```

Install: `pip install kokoro-onnx soundfile`

### Task 4 — Drop your model file

```
./models/qwen2.5-3b-instruct-q4_k_m.gguf
```

Update `MODEL_PATH` in `.env` if using a different filename. The `models/` folder is git-ignored — don't try to push the file.

---

## 🎙️ Rayan — What You Need To Implement

The backend WebSocket is fully ready. Here is the exact contract to build against.

### Connecting

```javascript
import { v4 as uuidv4 } from 'uuid';

const sessionId = uuidv4();
const ws = new WebSocket(`ws://localhost:8000/ws/chat?session_id=${sessionId}`);
```

### Recording and sending audio

```javascript
const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
const recorder = new MediaRecorder(stream, { mimeType: 'audio/webm' });

recorder.ondataavailable = (e) => {
    if (e.data.size > 0 && ws.readyState === WebSocket.OPEN) {
        ws.send(e.data);  // send Blob directly
    }
};

recorder.start(500);  // chunk every 500ms
```

### Receiving responses

```javascript
ws.onmessage = async (event) => {
    if (event.data instanceof Blob) {
        // TTS audio — play it
        const url = URL.createObjectURL(event.data);
        const audio = new Audio(url);
        await audio.play();
        URL.revokeObjectURL(url);

    } else {
        const msg = JSON.parse(event.data);

        if (msg.event === 'turn_complete') {
            updateTurnCounter(msg.turns_used, msg.turns_max);
            if (msg.status === 'ended') disableMicButton();
        }

        if (msg.event === 'error') {
            showError(msg.detail);
        }
    }
};
```

### Input sanitization (your scope)

```javascript
function sanitize(text) {
    const el = document.createElement('div');
    el.textContent = text;
    return el.innerHTML;
}
```

### Fetching welcome message on load

```javascript
const res = await fetch(`http://localhost:8000/session/welcome/${sessionId}`);
const { response } = await res.json();
displayMessage('assistant', response);
```

---

## 📊 Performance Benchmarks

> Fill in after Uwaid integrates the models.

```bash
# Step 1 — warmup (always do this first or first run will be slow)
curl -X POST http://localhost:8000/debug/warmup

# Step 2 — run benchmark
curl -X POST "http://localhost:8000/benchmark?runs=10"
```

| Metric | LLM only (ms) | Full pipeline STT+LLM+TTS (ms) |
|--------|:---:|:---:|
| Average | — | — |
| p50 | — | — |
| p95 | — | — |
| Min | — | — |
| Max | — | — |

Assignment target: **< 1000 ms** end-to-end per turn.

---

## 🦗 Load Testing

Simulates 4 concurrent users (3 REST + 1 WebSocket) as required by the assignment.

```bash
pip install locust websocket-client

# Interactive dashboard at http://localhost:8089
locust -f locustfile.py --host=http://localhost:8000
# Set: Users=4, Spawn rate=1, Duration=60s

# Headless — saves CSV to results/
locust -f locustfile.py --host=http://localhost:8000 \
       --users 4 --spawn-rate 1 --run-time 60s --headless \
       --csv=results/locust_report
```

---

## 🧪 Postman Collection

Import `daraz_assistant_a3.postman_collection.json` into Postman.

Set the collection variable `base_url = http://localhost:8000` then run requests **1 → 20 in order**. The session ID is auto-captured from request 6 (Welcome Message) and reused automatically for all subsequent requests.

Covers: health check, JWT auth cookies, full conversation flow, XSS sanitization check, payload rejection (413), rate limit trigger (429), and latency benchmarks with console output.

---

## 🐳 Docker Reference

```bash
# Start all services
docker compose up --build

# Rebuild backend only after code changes
docker compose up --build backend

# Live logs
docker compose logs -f backend
docker compose logs -f redis

# Stop
docker compose down

# Stop and wipe all volumes (fresh start)
docker compose down -v
```

---

## ⚙️ Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `REDIS_URL` | `redis://redis:6379/0` | Redis connection — auto-set in Docker Compose |
| `MODEL_PATH` | `/models/qwen2.5-3b-instruct-q4_k_m.gguf` | Uwaid: update to your actual filename |
| `FRONTEND_ORIGIN` | `http://localhost:3000` | Rayan: update to your Vercel URL before submission |
| `JWT_SECRET` | `CHANGE_ME` | Change to any long random string before deploying |
| `PYTHONUNBUFFERED` | `1` | Ensures Docker logs appear immediately |

---

## 🛑 .gitignore — Do Not Push These

```gitignore
models/
data/
.env
venv/
__pycache__/
results/
*.gguf
*.db
```