> **Commercial status:** Archived preview. This repository is kept for reference and is not an active commercial product.

# AI Meeting Notes → Linear/Notion Action Items

A FastAPI + pydantic-ai template that ingests meeting audio or raw transcript text, extracts structured action items via an LLM agent, and batch-creates Linear issues or Notion database rows — with field-level validation, deduplication, and retry logic built in.

---

## Overview

Otter.ai and Fireflies give you a wall of text. This template gives you tickets.

Drop in an audio file or paste a transcript. A pydantic-ai agent extracts every action item into a typed `ActionItem` model (owner, priority, due date, description). Those models are validated, deduplicated, and pushed to Linear or Notion in a single batch. No manual copy-paste, no missed follow-ups.

Built to extend the Email→Linear and GitHub→Linear templates — same patterns, new input modality.

---

## What It Does

1. **Accepts input** via HTTP: audio file upload (wav/mp3/m4a) or raw transcript text
2. **Transcribes audio** using OpenAI Whisper (local) or AssemblyAI (cloud) — configurable
3. **Runs a pydantic-ai agent** against the transcript to extract a list of `ActionItem` objects
4. **Validates every field** — owner, priority enum, due date format, description length — with Pydantic v2
5. **Deduplicates** against existing Linear issues or Notion rows using title similarity hashing
6. **Batch-creates** Linear issues or Notion database entries via their REST APIs
7. **Returns** a structured JSON response with created items, skipped duplicates, and any validation errors
8. **Retries** failed API calls with exponential backoff (configurable max attempts)

### ActionItem Schema

```python
class ActionItem(BaseModel):
    title: str                          # concise task title, max 100 chars
    description: str                    # full context from transcript
    owner: str | None                   # person mentioned as responsible
    priority: Literal["urgent", "high", "medium", "low"]
    due_date: date | None               # extracted or inferred date
    labels: list[str]                   # e.g. ["backend", "design"]
    source_quote: str                   # verbatim excerpt from transcript
```

---

## Prerequisites

- Python 3.11+
- An OpenAI API key (for the pydantic-ai agent and optionally Whisper)
- **Linear:** A Linear API key + Team ID (if using Linear output)
- **Notion:** A Notion integration token + Database ID (if using Notion output)
- **AssemblyAI** API key (optional — only if you prefer cloud transcription over local Whisper)
- `ffmpeg` installed locally if processing audio files (`brew install ffmpeg` / `apt install ffmpeg`)

---

## Setup

**1. Clone the repo and enter the directory**

```bash
git clone https://github.com/yourname/meeting-notes-to-linear.git
cd meeting-notes-to-linear
```

**2. Create and activate a virtual environment**

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
```

**3. Install dependencies**

```bash
pip install -r requirements.txt
```

**4. Copy the environment file and fill in your credentials**

```bash
cp .env.example .env
```

Open `.env` and set at minimum `OPENAI_API_KEY` plus either the Linear or Notion credentials (see [Configuration](#configuration) below).

**5. (Optional) Test your credentials**

```bash
python scripts/verify_credentials.py
```

This hits each configured API with a lightweight read request and reports which integrations are live.

**6. Start the server**

```bash
uvicorn app.main:app --reload --port 8000
```

The API is now running at `http://localhost:8000`. Interactive docs at `http://localhost:8000/docs`.

---

## Usage

### Submit a transcript (fastest path)

```bash
curl -X POST http://localhost:8000/process/transcript \
  -H "Content-Type: application/json" \
  -d '{
    "transcript": "Alice will own the API redesign by end of Q3. Bob needs to update the staging environment before Friday. We should urgently fix the payment bug — Charlie is on it.",
    "destination": "linear",
    "meeting_title": "Q3 Planning Sync"
  }'
```

### Upload an audio file

```bash
curl -X POST http://localhost:8000/process/audio \
  -F "file=@/path/to/meeting.mp3" \
  -F "destination=notion" \
  -F "meeting_title=Weekly Standup"
```

### Check job status (async jobs return a job_id)

```bash
curl http://localhost:8000/jobs/{job_id}
```

### List recently created action items

```bash
curl http://localhost:8000/action-items?limit=20&destination=linear
```

---

## API Endpoints

### `POST /process/transcript`

Accepts raw transcript text and returns extracted + created action items.

**Request body:**

```json
{
  "transcript": "string (required)",
  "destination": "linear | notion (required)",
  "meeting_title": "string (optional)",
  "default_priority": "medium (optional, default: medium)",
  "dry_run": false
}
```

**Response:**

```json
{
  "job_id": "uuid",
  "status": "completed",
  "created": [
    {
      "action_item": { "title": "...", "owner": "Alice", "priority": "high", "due_date": "2026-09-30" },
      "external_id": "LIN-142",
      "url": "https://linear.app/team/issue/LIN-142"
    }
  ],
  "skipped_duplicates": [],
  "validation_errors": [],
  "total_extracted": 3,
  "total_created": 3
}
```

---

### `POST /process/audio`

Accepts a multipart audio file upload. Transcribes with Whisper or AssemblyAI, then runs the same extraction pipeline.

**Form fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | yes | Audio file (wav, mp3, m4a, ogg, webm) |
| `destination` | string | yes | `linear` or `notion` |
| `meeting_title` | string | no | Used as context in the LLM prompt |
| `transcription_engine` | string | no | `whisper` (default) or `assemblyai` |
| `dry_run` | bool | no | Extract but don't create issues |

**Response:** Same schema as `/process/transcript`. Large audio files return a `job_id` and run asynchronously.

---

### `GET /jobs/{job_id}`

Poll for async job status.

```bash
curl http://localhost:8000/jobs/f47ac10b-58cc-4372-a567-0e02b2c3d479
```

**Response:**

```json
{
  "job_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "processing | completed | failed",
  "progress": "transcribing | extracting | creating_issues | done",
  "result": { ... },
  "error": null
}
```

---

### `GET /action-items`

Returns a paginated list of action items created in this session (in-memory or DB depending on config).

```bash
curl "http://localhost:8000/action-items?limit=10&offset=0&destination=linear"
```

---

### `GET /health`

Liveness check. Returns 200 with API connectivity status for each configured integration.

```bash
curl http://localhost:8000/health
```

```json
{
  "status": "ok",
  "integrations": {
    "openai": "connected",
    "linear": "connected",
    "notion": "not_configured"
  }
}
```

---

## Configuration

All configuration lives in `.env`. Copy `.env.example` to get started.

```env
# ── LLM ──────────────────────────────────────────────────────────────────────
OPENAI_API_KEY=sk-...                    # Required. Used for agent + Whisper transcription
OPENAI_MODEL=gpt-4o                      # Model for action item extraction (gpt-4o recommended)
OPENAI_TEMPERATURE=0.1                   # Lower = more consistent extraction

# ── Transcription ─────────────────────────────────────────────────────────────
TRANSCRIPTION_ENGINE=whisper             # "whisper" (local, free) or "assemblyai" (cloud)
ASSEMBLYAI_API_KEY=                      # Required only if TRANSCRIPTION_ENGINE=assemblyai
WHISPER_MODEL_SIZE=base                  # tiny | base | small | medium | large

# ── Linear ────────────────────────────────────────────────────────────────────
LINEAR_API_KEY=lin_api_...               # Settings → API → Personal API keys
LINEAR_TEAM_ID=abc123                    # Found in Linear URL or team settings
LINEAR_DEFAULT_ASSIGNEE_ID=             # Optional: Linear user ID for unowned items
LINEAR_PROJECT_ID=                       # Optional: attach issues to a specific project

# ── Notion ────────────────────────────────────────────────────────────────────
NOTION_TOKEN=secret_...                  # Settings → Integrations → New integration
NOTION_DATABASE_ID=abc123def456          # The 32-char ID from your database URL

# ── Deduplication ────────────────────────────────────────────────────────────
DEDUP_ENABLED=true                       # Skip items with >85% title similarity to existing
DEDUP_SIMILARITY_THRESHOLD=0.85          # 0.0–1.0, lower = more aggressive dedup
DEDUP_LOOKBACK_DAYS=30                   # How far back to check for duplicates

# ── Retry ────────────────────────────────────────────────────────────────────
MAX_RETRY_ATTEMPTS=3                     # Max retries for failed Linear/Notion API calls
RETRY_BACKOFF_BASE=2                     # Exponential backoff base in seconds

# ── Server ────────────────────────────────────────────────────────────────────
MAX_AUDIO_FILE_MB=100                    # Reject uploads over this size
ASYNC_THRESHOLD_SECONDS=30              # Jobs estimated longer than this run async
LOG_LEVEL=INFO                           # DEBUG | INFO | WARNING | ERROR
```

---

## Customization

### Change the extraction prompt

Edit `app/agents/extraction_agent.py`. The system prompt is a plain string — add domain-specific instructions, change what counts as an action item, or add new fields to the `ActionItem` model and the prompt simultaneously.

```python
SYSTEM_PROMPT = """
You are an expert meeting analyst. Extract every concrete action item...
# Add your custom instructions here
"""
```

### Add a new output destination

1. Create `app/integrations/your_tool.py` implementing the `DestinationAdapter` protocol (two methods: `check_duplicate`, `create_item`)
2. Register it in `app/integrations/__init__.py`
3. Add its config variables to `.env.example`

The router automatically discovers registered adapters — no other changes needed.

### Adjust priority mapping

Linear and Notion use different priority scales. Edit `app/integrations/linear.py` and `app/integrations/notion.py`:

```python
PRIORITY_MAP = {
    "urgent": 1,
    "high": 2,
    "medium": 3,
    "low": 4,
}
```

### Use a different transcription engine

Implement the `TranscriptionEngine` protocol in `app/transcription/` (one method: `async def transcribe(audio_bytes: bytes) -> str`), then set `TRANSCRIPTION_ENGINE` to your engine's registered name.

### Enable persistent storage

By default, job history is in-memory. Set `DATABASE_URL=postgresql+asyncpg://...` in `.env` to persist jobs and action items to Postgres. The models are SQLAlchemy-compatible and migrations are managed by Alembic (`alembic upgrade head`).

---

## License

MIT. Use it, modify it, sell products built with it.