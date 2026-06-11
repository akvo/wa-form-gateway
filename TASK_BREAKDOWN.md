# WhatsApp Data Collection Service - Task Breakdown

## Repository

| | |
|---|---|
| **Name** | wa-form-gateway |
| **URL** | https://github.com/akvo/wa-form-gateway |
| **Description** | Collect form data via WhatsApp conversations |

---

## Project Overview

A deployable Python service that enables data collection via WhatsApp conversations. The service fetches form definitions from backend platforms (starting with KoboToolbox), renders questions as WhatsApp messages, manages conversation state, and pushes completed submissions back to the source platform.

### Architecture

```
                    ┌─────────────────────────────┐
                    │   Meta WhatsApp Cloud API   │
                    │   (Webhooks + Send API)     │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼───────────────┐
                    │      WhatsApp Service       │
                    │  ┌───────────────────────┐  │
                    │  │  Conversation Engine  │  │
                    │  │  - Session Manager    │  │
                    │  │  - Question Renderer  │  │
                    │  │  - Response Validator │  │
                    │  └───────────────────────┘  │
                    │  ┌───────────────────────┐  │
                    │  │      Database         │  │
                    │  │  - Sessions           │  │
                    │  │  - Partial Responses  │  │
                    │  │  - Form Cache         │  │
                    │  └───────────────────────┘  │
                    └─────────────┬───────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
    ┌─────────▼─────────┐ ┌───────▼───────┐ ┌────────▼────────┐
    │   Kobo Adapter    │ │CommCare Adapter│ │  DHIS2 Adapter  │
    │   (Phase 1)       │ │   (Future)     │ │    (Future)     │
    └───────────────────┘ └───────────────┘ └─────────────────┘
```

---

## Phase 1: Foundation & Core Infrastructure

### 1.1 Project Setup & Configuration

**Task:** Initialize Python project with minimal dependencies and configuration management

**User AC:**
- Project can be cloned and set up with `pip install -e .`
- Configuration via environment variables or `.env` file
- Clear documentation for initial setup

**Tech AC:**
- Initialize Python package structure (`pyproject.toml` or `setup.py`)
- Core dependencies only:
  - `fastapi` + `uvicorn` (webhook server)
  - `httpx` (async HTTP client)
  - `sqlalchemy` (database ORM)
  - `pydantic` (validation/settings)
  - `python-dotenv` (env loading)
- Configuration schema:
  - Meta API credentials (phone number ID, access token, verify token)
  - Database URL (default SQLite)
  - Kobo API credentials (URL, token)
  - Logging level
- Create project structure:
  ```
  wa_form_gateway/
  ├── __init__.py
  ├── main.py              # FastAPI app entry
  ├── config.py            # Settings management
  ├── database/
  │   ├── models.py        # SQLAlchemy models
  │   └── session.py       # DB connection
  ├── whatsapp/
  │   ├── webhook.py       # Incoming message handler
  │   ├── sender.py        # Outgoing message API
  │   └── renderer.py      # Question → WhatsApp message
  ├── conversation/
  │   ├── engine.py        # Conversation flow logic
  │   ├── session.py       # Session state management
  │   └── validator.py     # Response validation
  └── adapters/
      ├── base.py          # Abstract adapter interface
      └── kobo.py          # Kobo implementation
  ```

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Keep dependencies minimal for easier deployment
- SQLite for development, Postgres-ready for production

---

### 1.2 Database Models & Session Management

**Task:** Design and implement database schema for conversation state tracking

**User AC:**
- Service remembers where each user left off in a form
- Partial responses are preserved if user disconnects
- Multiple users can fill forms simultaneously

**Tech AC:**
- Database models:
  ```python
  Session:
    - id (UUID)
    - phone_number (string, indexed)
    - form_id (string)
    - adapter_type (string, e.g., "kobo")
    - current_question_index (int)
    - status (enum: ACTIVE, COMPLETED, EXPIRED, CANCELLED)
    - started_at (datetime)
    - updated_at (datetime)
    - expires_at (datetime)

  Response:
    - id (UUID)
    - session_id (FK)
    - question_id (string)
    - question_type (string)
    - value (JSON)
    - answered_at (datetime)

  FormCache:
    - id (UUID)
    - adapter_type (string)
    - form_id (string)
    - form_definition (JSON)
    - cached_at (datetime)
    - expires_at (datetime)
  ```
- Session manager class:
  - `get_or_create_session(phone_number, form_id)`
  - `save_response(session_id, question_id, value)`
  - `advance_session(session_id)`
  - `complete_session(session_id)`
  - `expire_stale_sessions()`
- Implement session timeout (configurable, default 24h)

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Phone numbers must be normalized (E.164 format)
- Consider GDPR: add ability to purge user data
- Index on phone_number for fast lookups

---

## Phase 2: WhatsApp Integration (Meta Cloud API)

### 2.1 Webhook Endpoint & Message Receiver

**Task:** Implement Meta WhatsApp Cloud API webhook for receiving messages

**User AC:**
- Service can be registered as WhatsApp webhook
- Incoming text messages are received and logged
- Incoming button/list replies are captured

**Tech AC:**
- `GET /webhook` - Verification endpoint (Meta challenge)
- `POST /webhook` - Message receiver
- Parse webhook payload:
  - Extract sender phone number
  - Extract message type (text, button_reply, list_reply)
  - Extract message content/selection
- Handle webhook verification challenge
- Implement signature verification (X-Hub-Signature-256)
- Return 200 quickly, process async (Meta requires fast response)
- Message types to handle:
  - `text` - Free text response
  - `interactive.button_reply` - Button selection
  - `interactive.list_reply` - List selection
- Logging for debugging

**Estimate:** 1 day (with AI-assisted development)

**Risk / Considerations:**
- Meta requires HTTPS with valid certificate
- Webhook must respond within 20 seconds
- Must handle duplicate webhook deliveries (idempotency)
- Rate limits: 80 messages/second (Business API)

---

### 2.2 Message Sender & Question Renderer

**Task:** Implement outgoing message API for different question types

**User AC:**
- Text questions appear as plain messages
- Single-select questions appear as buttons (≤3 options) or lists (>3 options)
- Number questions show input hint
- Users receive clear instructions

**Tech AC:**
- WhatsApp message types to implement:
  - `text` - For text/number questions and instructions
  - `interactive.button` - For 1-3 options (single select)
  - `interactive.list` - For 4+ options (single select)
- Question renderer:
  ```python
  def render_question(question: Question) -> WhatsAppMessage:
      if question.type == "note":
          return TextMessage(question.label)
      elif question.type == "text":
          return TextMessage(f"{question.label}\n\n_Type your answer:_")
      elif question.type == "integer":
          return TextMessage(f"{question.label}\n\n_Enter a number:_")
      elif question.type == "select_one":
          if len(question.options) <= 3:
              return ButtonMessage(question.label, question.options)
          else:
              return ListMessage(question.label, question.options)
  ```
- Meta Graph API client:
  - `POST /{phone_number_id}/messages`
  - Handle rate limiting (exponential backoff)
  - Handle errors (invalid number, blocked, etc.)
- Message templates for:
  - Welcome message
  - Question prompts
  - Validation errors
  - Completion confirmation

**Estimate:** 1 day (with AI-assisted development)

**Risk / Considerations:**
- Button text max 20 characters
- List item text max 24 characters
- Message body max 1024 characters
- May need to truncate long option labels
- WhatsApp buttons limited to 3 options maximum

---

### 2.3 Response Validation

**Task:** Validate user responses based on question type

**User AC:**
- Invalid responses get helpful error messages
- Users can retry after invalid input
- Clear feedback on what went wrong

**Tech AC:**
- Validators per question type:
  - `text`: Non-empty (if required)
  - `integer`: Must be valid integer, optional range
  - `decimal`: Must be valid decimal
  - `select_one`: Must match an option ID
  - `select_multiple`: All selections must be valid
- Validation response:
  ```python
  ValidationResult:
    - is_valid: bool
    - normalized_value: Any
    - error_message: Optional[str]
  ```
- Re-send question with error hint on failure
- Track retry count (optional: max retries before skip/cancel)

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Users may send voice messages (not supported initially)
- Users may send images (not supported initially)
- Handle edge cases: empty messages, emojis in numbers

---

## Phase 3: Conversation Engine

### 3.1 Conversation Flow Controller

**Task:** Implement the core conversation state machine

**User AC:**
- Starting a form sends the first question
- Each valid answer advances to next question
- Form completion triggers submission
- Users can restart or cancel mid-form

**Tech AC:**
- Conversation engine class:
  ```python
  ConversationEngine:
    - start_form(phone_number, form_id) -> sends first question
    - handle_message(phone_number, message) -> processes response
    - get_next_question(session) -> Question or None
    - submit_form(session) -> sends to adapter
  ```
- Flow:
  1. Receive message
  2. Look up active session for phone number
  3. If no session: show form selection or error
  4. If session active: validate response
  5. If valid: save response, get next question
  6. If last question: compile and submit
  7. Send appropriate message
- Command handling:
  - `/start` or `hi` - Show available forms or welcome
  - `/cancel` - Cancel current session
  - `/restart` - Restart current form
- Handle "out of band" messages during form fill

**Estimate:** 1-1.5 days (with AI-assisted development)

**Risk / Considerations:**
- Must handle concurrent messages from same user
- Race conditions on session updates
- User might message from different device

---

### 3.2 Form Definition Parser (Internal Format)

**Task:** Create internal form representation that adapters translate to

**User AC:**
- Forms from any adapter work with the conversation engine
- Consistent behavior regardless of source platform

**Tech AC:**
- Internal form schema:
  ```python
  Form:
    - id: str
    - title: str
    - questions: List[Question]

  Question:
    - id: str
    - type: QuestionType (text, integer, decimal, select_one, select_multiple, note)
    - label: str
    - hint: Optional[str]
    - required: bool
    - options: Optional[List[Option]]  # for select types
    - constraints: Optional[Constraints]

  Option:
    - id: str
    - label: str

  Constraints:
    - min: Optional[number]
    - max: Optional[number]
    - regex: Optional[str]
  ```
- Adapter interface:
  ```python
  class BaseAdapter(ABC):
      def fetch_form(self, form_id: str) -> Form
      def submit(self, form_id: str, responses: dict) -> SubmissionResult
  ```

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Not all Kobo question types can be mapped to WhatsApp
- Complex skip logic not supported in Phase 1

---

## Phase 4: Kobo Adapter

### 4.1 Kobo API Client

**Task:** Implement KoboToolbox API integration for form fetching

**User AC:**
- Service can list available forms from Kobo
- Service can fetch form structure/questions
- Kobo credentials configured once

**Tech AC:**
- Kobo API v2 endpoints:
  - `GET /api/v2/assets/` - List forms
  - `GET /api/v2/assets/{uid}/` - Form metadata
  - `GET /api/v2/assets/{uid}/?format=json` - Form structure
- Authentication: Token-based (`Authorization: Token {api_token}`)
- Parse Kobo form structure:
  - Extract questions from `content.survey`
  - Extract choices from `content.choices`
  - Map Kobo types to internal types:
    - `text` → `text`
    - `integer` → `integer`
    - `decimal` → `decimal`
    - `select_one` → `select_one`
    - `select_multiple` → `select_multiple`
    - `note` → `note`
    - Others → skip or `text` fallback
- Cache form definitions (avoid repeated API calls)

**Estimate:** 1 day (with AI-assisted development)

**Risk / Considerations:**
- Kobo API rate limits
- Large forms may have many questions
- XLSForm features not all supported (calculations, skip logic, repeats)

---

### 4.2 Kobo Submission Push

**Task:** Submit completed form data back to KoboToolbox

**User AC:**
- Completed WhatsApp forms appear in Kobo data
- Submissions include metadata (timestamp, source)
- Failed submissions are retried

**Tech AC:**
- Kobo submission endpoint:
  - `POST /api/v2/assets/{uid}/data/` (JSON submission)
  - Or: OpenRosa submission endpoint `/submission`
- Format submission payload:
  ```json
  {
    "question_name_1": "value",
    "question_name_2": 123,
    "_submitted_by": "whatsapp",
    "_submission_time": "2025-01-01T00:00:00Z",
    "meta/phone_number": "+1234567890"
  }
  ```
- Handle submission response:
  - Success: Mark session complete, notify user
  - Failure: Log error, retry queue, notify user
- Implement retry mechanism (3 attempts with backoff)
- Store submission ID from Kobo for reference

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Kobo may reject submissions if required fields missing
- Validation rules on Kobo side may fail
- Need to handle Kobo downtime gracefully

---

## Phase 5: Deployment & Operations

### 5.1 Docker Containerization

**Task:** Create Docker setup for easy deployment

**User AC:**
- Service can be deployed with `docker-compose up`
- Configuration via environment variables
- Persistent data survives container restarts

**Tech AC:**
- `Dockerfile`:
  - Python 3.11+ base image
  - Install dependencies
  - Run with uvicorn
- `docker-compose.yml`:
  - App service
  - PostgreSQL service (optional, SQLite default)
  - Volume for database persistence
- Health check endpoint (`/health`)
- Graceful shutdown handling

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- SQLite in Docker needs volume mount
- Consider memory limits for container

---

### 5.2 Webhook Registration & HTTPS Setup

**Task:** Document and/or automate Meta webhook setup

**User AC:**
- Clear instructions to register webhook with Meta
- Service works with common reverse proxies (nginx, Cloudflare)

**Tech AC:**
- Documentation:
  - Meta Business App setup
  - WhatsApp Business API access
  - Webhook URL registration
  - Verify token configuration
- Optional CLI command: `python -m wa_form_gateway verify`
- Nginx/Caddy example configs
- Cloudflare tunnel option for development

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Meta requires verified business for production
- Webhook URL must be publicly accessible HTTPS
- Development/testing requires ngrok or similar
- External: Meta Business verification may take 1-5 business days

---

### 5.3 Logging, Monitoring & Error Handling

**Task:** Implement operational observability

**User AC:**
- Errors are logged with context
- Can troubleshoot failed conversations
- Basic metrics available

**Tech AC:**
- Structured logging (JSON format option)
- Log levels: DEBUG, INFO, WARNING, ERROR
- Log contexts:
  - Phone number (hashed for privacy)
  - Session ID
  - Form ID
  - Error details
- Error handling:
  - Graceful degradation
  - User-friendly error messages
  - Admin notifications (optional webhook)
- Health endpoint with:
  - Database connectivity
  - Meta API connectivity
  - Kobo API connectivity

**Estimate:** 0.5 day (with AI-assisted development)

**Risk / Considerations:**
- Don't log full phone numbers (privacy)
- Log rotation for disk space

---

## Phase 6: Testing & Documentation

### 6.1 Test Suite

**Task:** Implement automated tests

**User AC:**
- Core functionality is tested
- Changes don't break existing features

**Tech AC:**
- Unit tests:
  - Form parser
  - Response validators
  - Question renderer
- Integration tests:
  - Conversation flow (mocked WhatsApp)
  - Kobo adapter (mocked API)
- Test fixtures:
  - Sample Kobo forms
  - Sample webhook payloads
- pytest configuration
- CI pipeline (GitHub Actions)

**Estimate:** 1 day (with AI-assisted development)

**Risk / Considerations:**
- Meta webhook testing requires mocks
- End-to-end testing with real WhatsApp is manual

---

### 6.2 Documentation

**Task:** Create user and developer documentation

**User AC:**
- New users can deploy and configure
- Developers can add new adapters

**Tech AC:**
- README.md:
  - Quick start
  - Configuration reference
  - Deployment options
- ADAPTERS.md:
  - How to create new adapter
  - Interface specification
  - Example adapter
- API.md:
  - Internal API endpoints
  - Admin endpoints (if any)
- TROUBLESHOOTING.md:
  - Common issues
  - Debug mode

**Estimate:** 0.5 day (with AI-assisted development)

---

## Summary & Estimates

> **Note:** Estimates assume AI-assisted development (Claude Code). Manual development would require approximately 2-2.5x the listed time.

| Phase | Task | Estimate |
|-------|------|----------|
| **1. Foundation** | | |
| 1.1 | Project Setup & Configuration | 0.5 day |
| 1.2 | Database Models & Session Management | 0.5 day |
| **2. WhatsApp Integration** | | |
| 2.1 | Webhook Endpoint & Message Receiver | 1 day |
| 2.2 | Message Sender & Question Renderer | 1 day |
| 2.3 | Response Validation | 0.5 day |
| **3. Conversation Engine** | | |
| 3.1 | Conversation Flow Controller | 1-1.5 days |
| 3.2 | Form Definition Parser | 0.5 day |
| **4. Kobo Adapter** | | |
| 4.1 | Kobo API Client | 1 day |
| 4.2 | Kobo Submission Push | 0.5 day |
| **5. Deployment** | | |
| 5.1 | Docker Containerization | 0.5 day |
| 5.2 | Webhook & HTTPS Setup | 0.5 day* |
| 5.3 | Logging & Monitoring | 0.5 day |
| **6. Testing & Docs** | | |
| 6.1 | Test Suite | 1 day |
| 6.2 | Documentation | 0.5 day |
| | | |
| **TOTAL (Development)** | | **~10 days** |
| **External/Blocking** | Meta Business verification, testing with real WhatsApp | **+2-5 days*** |

\* *External dependencies (Meta verification, webhook setup, real device testing) may add calendar time regardless of development speed.*

---

## Limitations & Known Constraints

### WhatsApp Platform Limitations
1. **Button limit**: Maximum 3 buttons per message - options >3 must use list
2. **List limit**: Maximum 10 items per section, 10 sections max
3. **Character limits**: Button text 20 chars, list item 24 chars, body 1024 chars
4. **No rich input**: Cannot request date pickers, file uploads, or signatures
5. **24-hour window**: Can only message users who messaged within 24 hours (unless using templates)
6. **Template approval**: Proactive messages require pre-approved templates
7. **Media not supported** (Phase 1): Voice messages, images, documents ignored

### Kobo Form Limitations
1. **Question types not supported (Phase 1)**:
   - `geopoint`, `geotrace`, `geoshape` (location)
   - `image`, `audio`, `video`, `file` (media)
   - `date`, `time`, `datetime` (no native picker, text fallback possible)
   - `barcode` (no scanner)
   - `range` (slider)
   - `rank` (ranking)
2. **No skip logic**: All questions shown linearly regardless of conditions
3. **No calculations**: Calculated fields not evaluated
4. **No repeating groups**: Cannot dynamically repeat question sets
5. **No cascading selects**: Filtered option lists not supported
6. **No groups**: Questions flattened, no grouping in conversation

### Operational Limitations
1. **Single form per session**: User completes one form at a time
2. **No concurrent forms**: User cannot fill multiple forms simultaneously
3. **No edit after submit**: Cannot modify submitted responses
4. **Session timeout**: Incomplete sessions expire (default 24h)
5. **No offline**: Requires constant internet connectivity
6. **Single language**: Form presented in one language (no runtime switch)

### Future Enhancements (Not in Scope)
- [ ] Skip logic / conditional questions
- [ ] Date/time questions with validation
- [ ] Multi-language support
- [ ] Media question types (photo capture via WhatsApp)
- [ ] Repeating groups
- [ ] Form assignment (push specific forms to users)
- [ ] Bulk messaging / campaigns
- [ ] Analytics dashboard
- [ ] CommCare adapter
- [ ] DHIS2 adapter
- [ ] ODK Central adapter

---

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Meta API changes | High | Low | Pin API version, monitor changelog |
| Kobo API changes | Medium | Low | Pin API version, integration tests |
| WhatsApp rate limiting | Medium | Medium | Implement backoff, queue messages |
| Session data loss | High | Low | Database backups, transaction safety |
| User privacy concerns | High | Medium | Hash phone numbers in logs, data retention policy |
| Webhook downtime | High | Medium | Health monitoring, alerting |
| Long forms fatigue | Medium | High | Document form design best practices |
| Invalid data to Kobo | Medium | Medium | Pre-submission validation |

---

## Dependencies & Prerequisites

### Meta/WhatsApp Setup
- [ ] Meta Business Account
- [ ] Meta Developer App with WhatsApp Product
- [ ] WhatsApp Business Account
- [ ] Verified Business (for production)
- [ ] Phone number registered with WhatsApp Business API
- [ ] Access Token with `whatsapp_business_messaging` permission

### Kobo Setup
- [ ] KoboToolbox account (self-hosted or kobotoolbox.org)
- [ ] API token generated
- [ ] At least one deployed form for testing

### Infrastructure
- [ ] Server with public IP or domain
- [ ] Valid SSL certificate (Let's Encrypt or similar)
- [ ] Python 3.11+ environment
- [ ] PostgreSQL (production) or SQLite (development)

---

*Document Version: 1.0*
*Created: 2026-06-11*
