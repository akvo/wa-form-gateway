# Project Context

> Quick reference for continuing development in future sessions.

**Repo:** https://github.com/akvo/wa-form-gateway

---

## What This Is

A **WhatsApp-based data collection service** that:
1. Fetches forms from backend platforms (Kobo, future: CommCare, DHIS2)
2. Renders questions as WhatsApp messages (conversational, one at a time)
3. Tracks session state (which question user is on, partial responses)
4. Pushes completed submissions back to the source platform

## Key Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture | Deployable service (not library) | Needs database for session state, webhook endpoint |
| Core focus | WhatsApp is the center | Adapters (Kobo, etc.) plug into WhatsApp service |
| Language | Python | User preference |
| Dependencies | Minimal | FastAPI, SQLAlchemy, httpx, pydantic |
| Database | SQLite (dev) / Postgres (prod) | Simple default, scalable option |
| First adapter | KoboToolbox | User's primary use case |

## Tech Stack (Planned)

```
fastapi          # Webhook server
uvicorn          # ASGI server
httpx            # Async HTTP client (Meta API, Kobo API)
sqlalchemy       # Database ORM
pydantic         # Settings & validation
python-dotenv    # Environment config
```

## Project Structure (Planned)

```
wa_form_gateway/
├── main.py              # FastAPI entry
├── config.py            # Settings
├── database/
│   ├── models.py        # Session, Response, FormCache
│   └── session.py       # DB connection
├── whatsapp/
│   ├── webhook.py       # Receive messages
│   ├── sender.py        # Send messages
│   └── renderer.py      # Question → WhatsApp format
├── conversation/
│   ├── engine.py        # Flow control
│   ├── session.py       # State management
│   └── validator.py     # Response validation
└── adapters/
    ├── base.py          # Abstract interface
    └── kobo.py          # Kobo implementation
```

## Phase 1 Scope (MVP)

**Supported:**
- Question types: text, integer, decimal, select_one, select_multiple, note
- Linear flow (all questions in order)
- Single form per session
- Basic validation (required, type checking)

**Not supported (Phase 1):**
- Skip logic / conditions
- Calculations
- Repeating groups
- Media questions (photo, audio, geo)
- Multi-language

## External Requirements

- Meta Business Account + WhatsApp Business API access
- KoboToolbox account with API token
- HTTPS endpoint (for webhook)

## Files

- `TASK_BREAKDOWN.md` - Full Asana task breakdown with estimates
- `PROJECT_CONTEXT.md` - This file

## Next Step

Start Phase 1.1: Project Setup & Configuration
- Initialize Python package
- Set up FastAPI skeleton
- Create config management
