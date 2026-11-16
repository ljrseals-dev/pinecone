# Pinecone Agent Harness (v2)

Production-oriented **agentic chatbot** with:

- Pinecone vector memory (session-isolated namespaces)
- Mitigated memory (summarization-ready, soft-delete pruning, near-duplicate detection)
- Structured agent loop (tool_call / final_answer)
- Restricted tool execution
- FastAPI REST API

---

## Quick Start

### 1. Clone / unzip & install

```bash
cd pinecone-agent-harness
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env and set:
#   PINECONE_API_KEY=...
#   OPENAI_API_KEY=...
```

### 3. Run the API

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Open the interactive docs: http://localhost:8000/docs

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET`  | `/health` | Health check |
| `POST` | `/sessions` | Create a new session |
| `GET`  | `/sessions/{id}` | Check session exists |
| `POST` | `/chat` | Main chatbot endpoint |
| `GET`  | `/sessions/{id}/memory` | Inspect retrieved memories |
| `DELETE` | `/sessions/{id}` | Drop in-memory session |

### Example – Chat

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What are the key differences between Pinecone serverless and pod-based indexes? Store the comparison and summarize it."
  }'
```

Response shape:

```json
{
  "session_id": "session-a1b2c3d4e5",
  "answer": "...",
  "status": "completed",
  "steps": 3,
  "tokens_used": 1842
}
```

Reuse the same `session_id` on subsequent calls to continue the conversation with the same memory namespace.

---

## Architecture Overview

```
Client
  │
  ▼
FastAPI  (/chat, /sessions, /memory)
  │
  ▼
AgentHarness  (structured loop: tool_call | final_answer)
  │
  ├── restricted tool runtime (allow-listed builtins)
  │
  └── PineconeMemory
        ├── content-addressable IDs + near-dupe detection
        ├── soft-delete + prune
        └── token-budgeted context assembly
              │
              ▼
         Pinecone Serverless (per-session namespace)
```

---

## Key Design Decisions

| Concern | Approach |
|---------|----------|
| Memory isolation | One Pinecone namespace per `session_id` |
| Idempotent writes | Content-hash vector IDs |
| Near-duplicates | Cosine ≥ 0.95 → bump access instead of insert |
| Deletion | Soft-delete (`deleted_at`) first |
| Context control | Hard token budget + composition logging |
| Tool safety | Restricted builtins + output size cap + sanitization |
| Termination | JSON schema: `tool_call` or `final_answer` only |
| Cost control | `max_steps` + `max_run_tokens` circuit breakers |

---

## Project Layout

```
pinecone-agent-harness/
├── app/
│   ├── __init__.py
│   ├── config.py          # pydantic-settings
│   ├── memory.py          # PineconeMemory
│   ├── harness.py         # AgentHarness + restricted executor
│   └── main.py            # FastAPI routes
├── tests/
├── .env.example
├── requirements.txt
├── README.md
└── .gitignore
```

---

## Production Notes

- Replace the in-memory `SESSIONS` dict with Redis or a database.
- Run tool execution inside a real sandbox (gVisor / Firecracker / WASM) for untrusted code.
- Add authentication (API keys / JWT) in front of `/chat`.
- Use Pinecone `fetch_by_metadata` with pagination for large-scale pruning.
- Enable OpenTelemetry for traces and the metrics defined in the design review.
- Set `CORS_ORIGINS` to your frontend domain(s).

---

## License

MIT
