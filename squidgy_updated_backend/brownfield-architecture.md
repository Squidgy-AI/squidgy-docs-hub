# Squidgy Backend -- Brownfield Architecture Document

**Audience:** Human developers onboarding to the project AND AI coding agents working on this codebase.
**Last verified against codebase:** 2026-02-22

---

## Change Log

| Date       | Author        | Change                                      |
|------------|---------------|----------------------------------------------|
| 2026-02-22 | Claude Opus 4.6 | Initial brownfield architecture document created |

---

## Quick Reference

| Item                  | Value                                                              |
|-----------------------|--------------------------------------------------------------------|
| Language              | Python 3.12 (3.13 INCOMPATIBLE)                                   |
| Framework             | FastAPI 0.115.8 (async ASGI)                                      |
| Server                | Gunicorn 21.2.0 + Uvicorn 0.34.0 workers                          |
| Hosting               | Heroku (512MB dyno)                                                |
| Primary database      | Supabase PostgreSQL (auth, profiles, chat history, credentials)    |
| Vector database       | Neon PostgreSQL with pgvector (semantic search, 1536-dim embeddings)|
| Repo                  | `https://github.com/Squidgy-AI/Squidgy_backend_heroku`            |
| Monolith file         | `main.py` -- 9,338 lines, 74 endpoint decorators                  |
| Total endpoints       | ~75 in main.py + additional in mounted routers                     |
| Procfile command      | `chmod +x install_chromium.sh && ./install_chromium.sh && WEB_CONCURRENCY=2 gunicorn main:app --config gunicorn_config.py --preload` |
| Local run             | `uvicorn main:app --host 0.0.0.0 --port 8000`                     |
| Health check          | `GET /` or `GET /health`                                           |

---

## 1. System Overview

The Squidgy backend is a Python FastAPI monolith deployed on Heroku. It serves as the API layer for the Squidgy AI platform, handling user authentication, chat orchestration (via N8N webhooks), website analysis, GoHighLevel (GHL) CRM automation, Facebook social media integration, knowledge base management with vector search, file processing, and template management.

The defining architectural characteristic is that nearly all logic lives in a single `main.py` file of 9,338 lines. Some newer functionality has been extracted into `routes/` modules and separate directories, but the bulk remains monolithic.

```
                    +-------------------+
                    |   Heroku Dyno     |
                    |   (512MB limit)   |
                    +--------+----------+
                             |
                    +--------v----------+
                    |    Gunicorn        |
                    |  (2 workers, 29s) |
                    +--------+----------+
                             |
                    +--------v----------+
                    |  Uvicorn Worker    |
                    |  (FastAPI/ASGI)    |
                    +--------+----------+
                             |
          +------------------+------------------+
          |                  |                  |
  +-------v------+  +-------v------+  +-------v------+
  |   main.py    |  |   routes/    |  | GHL_Marketing|
  | (9338 lines) |  | (3 modules)  |  | (2 modules)  |
  +--------------+  +--------------+  +--------------+
          |
  +-------+-------+-------+-------+-------+
  |       |       |       |       |       |
Supa-   Neon    N8N    GHL    Perplexity  Playwright
base    (vec)  (wh)   API      AI        (browser)
```

---

## 2. Tech Stack

| Category         | Technology          | Version   | Purpose                                      |
|------------------|---------------------|-----------|----------------------------------------------|
| Framework        | FastAPI             | 0.115.8   | Async API framework                          |
| WSGI Server      | Gunicorn            | 21.2.0    | Process manager, 2 workers                   |
| ASGI Server      | Uvicorn             | 0.34.0    | Async server under Gunicorn                  |
| Language         | Python              | 3.12      | **3.13 has known incompatibilities**          |
| Auth/Storage DB  | Supabase            | 2.27.2    | Auth, PostgreSQL, file storage               |
| Vector DB        | Neon PostgreSQL     | asyncpg 0.29.0 | pgvector for semantic search            |
| Browser          | Playwright          | 1.53.0    | Chromium-only, screenshots, GHL automation   |
| PDF parsing      | pdfplumber          | 0.10.3    | Primary PDF text extraction                  |
| PDF fallback     | PyPDF2              | 3.0.1     | Secondary PDF extraction                     |
| DOCX parsing     | python-docx         | 1.1.0     | Word document extraction                     |
| AI (embeddings)  | OpenRouter           | via HTTP  | text-embedding-3-small (1536 dimensions)     |
| AI (search)      | Perplexity          | via HTTP  | Web search and reasoning                     |
| AI SDK           | OpenAI              | 1.58.1    | Installed but currently commented out         |
| SMS              | Twilio              | 8.10.0    | SMS messaging webhook                        |
| HTTP client      | httpx               | 0.28.1    | Async HTTP calls                             |
| HTTP client      | aiohttp             | 3.10.5    | Additional async HTTP                        |
| HTTP client      | requests            | 2.32.3    | Sync HTTP calls                              |
| Validation       | Pydantic            | 2.12.5    | Request/response models                      |
| Image processing | Pillow              | 11.1.0    | Image manipulation                           |
| Web scraping     | BeautifulSoup4      | 4.12.3    | HTML parsing                                 |
| Security         | bandit, safety      | 1.7.5, 2.3.5 | Security scanning (MCP feature)          |
| Monitoring       | psutil              | 5.9.6     | System resource monitoring                   |
| Crypto           | cryptography        | 42.0.8    | Auth token handling                          |

---

## 3. Project Structure

```
squidgy_updated_backend/
|
|-- main.py                          # 9,338 lines. THE monolith. 74 endpoint decorators.
|
|-- routes/                          # Extracted route modules (newer code)
|   |-- __init__.py
|   |-- knowledge_base.py           # 844 lines. KB CRUD + semantic search via Neon.
|   |-- ghl_media.py                # 251 lines. GoHighLevel media endpoints.
|   |-- templated_io.py             # 506 lines. Templated.io design template management.
|
|-- GHL_Marketing/                   # Social media OAuth routers
|   |-- social_facebook.py          # 18,537 lines. Facebook OAuth + page management.
|   |-- social_instagram.py         # 18,785 lines. Instagram OAuth + posting.
|
|-- Website/                         # Web scraping utilities
|   |-- __init__.py
|   |-- web_scrape.py               # Screenshot capture, favicon extraction
|   |-- web_analysis.py             # Website content analysis, color extraction
|
|-- Tools/                           # GHL API wrappers (EXCLUDED from Heroku slug)
|   |-- GHL/
|   |   |-- Appointments/
|   |   |-- Calendars/
|   |   |-- Contacts/
|   |   |-- Sub_Accounts/
|   |   |-- Users/
|   |   |-- access_token.py
|   |   |-- environment/
|   |-- Website/
|   |-- SolarWebsiteAnalysis/
|   |-- SolarWebsiteAnalysis.py
|
|-- Database/                        # SQL schema files (10 files, excluded from slug)
|   |-- audit_data_capture.sql
|   |-- create_sql.sql
|   |-- essential_data_strategy.sql
|   |-- fix_embedding_dimensions.sql
|   |-- fix_search_function.sql
|   |-- mcp_tables.sql
|   |-- optimized_schema.sql
|   |-- prevent_chat_duplicates.sql
|
|-- mcp/                             # Model Context Protocol server
|   |-- server.py                    # MCPGateway class, mounted at /api/v1/mcp
|   |-- config.py, config_loader.py
|   |-- models.py, registry.py
|   |-- bridges/, custom/, external/, security/
|
|-- migrations/                      # Database migration scripts
|-- scripts/                         # Utility scripts
|-- static/                          # Screenshots and favicons (runtime-generated)
|
|-- background_text_processor.py     # Async file text extraction (background tasks)
|-- file_processing_service.py       # File upload + processing pipeline
|-- invitation_handler.py            # Email invitation sending
|-- email_validation.py              # Email format validation
|-- web_analysis_client.py           # WebAnalysisClient wrapper
|-- facebook_oauth_interceptor.py    # Facebook OAuth interception pattern
|-- company_analyzer.py              # Company/business analysis
|-- database.py                      # Database connection helpers
|
|-- gunicorn_config.py               # Gunicorn server configuration
|-- Procfile                         # Heroku process definition
|-- app.json                         # Heroku manifest (buildpacks, addons, env)
|-- requirements.txt                 # Python dependencies (production)
|-- runtime.txt                      # Python version: python-3.12
|-- Aptfile                          # System-level apt packages for Heroku
|-- install_chromium.sh              # Pre-start script: installs Playwright Chromium
|-- .slugignore                      # 70+ exclusion patterns to shrink Heroku slug
```

### Files Imported by main.py at Startup

```python
from Website.web_scrape import capture_website_screenshot, get_website_favicon_async
from Website.web_analysis import analyze_website, extract_colors_from_website
from invitation_handler import InvitationHandler
from file_processing_service import FileProcessingService
from background_text_processor import get_background_processor, initialize_background_processor
from web_analysis_client import WebAnalysisClient
```

### Routers Mounted at Startup (try/except guarded)

```python
app.include_router(ghl_media_router)                               # from routes.ghl_media
app.include_router(knowledge_base_router)                          # from routes.knowledge_base
app.include_router(social_facebook_router)                         # from GHL_Marketing.social_facebook
app.include_router(social_instagram_router)                        # from GHL_Marketing.social_instagram
app.include_router(mcp_gateway.router, prefix="/api/v1/mcp")      # from mcp.server
app.include_router(templated_router)                               # from routes.templated_io
```

All router mounts are wrapped in `try/except` so missing modules degrade gracefully.

---

## 4. API Endpoints

### 4.1 Full Endpoint Map (main.py -- 74 decorators)

**Health & Utility**

| Method | Path                     | Line  | Purpose                        |
|--------|--------------------------|-------|--------------------------------|
| GET    | `/`                      | 779   | Health check (basic)           |
| GET    | `/health`                | 783   | Health check (detailed)        |
| GET    | `/logs`                  | 1857  | Application log viewer         |

**N8N Integration**

| Method | Path                                        | Line  | Purpose                              |
|--------|---------------------------------------------|-------|--------------------------------------|
| POST   | `/n8n/agent/query`                          | 897   | Agent knowledge base query           |
| POST   | `/n8n/agent/query/wrapper`                  | 1140  | N8N-specific KB query wrapper        |
| POST   | `/n8n/agent/refresh_kb`                     | 1186  | Refresh agent knowledge base         |
| POST   | `/n8n_main_req/{agent_name}/{session_id}`   | 1213  | Main N8N agent request               |
| POST   | `/n8n_main_req_stream`                      | 1321  | Streaming N8N request                |

**Chat & Real-time**

| Method    | Path                                | Line  | Purpose                              |
|-----------|-------------------------------------|-------|--------------------------------------|
| WebSocket | `/ws/{user_id}/{session_id}`        | 1891  | Real-time chat (ping/pong keepalive) |
| POST      | `/api/stream`                       | 1348  | Receive SSE stream updates           |
| GET       | `/chat-history`                     | 1809  | Retrieve chat history                |
| POST      | `/api/webhooks/ghl/messages`        | 1395  | GHL inbound message webhook          |
| POST      | `/api/webhooks/twilio/sms`          | 9194  | Twilio SMS webhook                   |

**Auth & Users**

| Method | Path                              | Line  | Purpose                      |
|--------|-----------------------------------|-------|------------------------------|
| POST   | `/api/auth/signup`                | 2381  | User registration            |
| POST   | `/api/auth/confirm-signup`        | 2505  | Confirm signup               |
| POST   | `/api/auth/confirm-email` (POST)  | 2533  | Email confirmation (POST)    |
| GET    | `/api/auth/confirm-email` (GET)   | 2636  | Email confirmation (GET)     |
| GET    | `/confirm-email`                  | 2669  | Email confirmation HTML page |
| POST   | `/api/auth/reset-password`        | 2267  | Password reset email         |
| POST   | `/api/auth/update-password`       | 2319  | Update password              |
| POST   | `/api/send-invitation-email`      | 2127  | Send invitation              |

**Notifications**

| Method | Path                                          | Line  | Purpose                  |
|--------|-----------------------------------------------|-------|--------------------------|
| GET    | `/api/notifications/{user_id}`                | 1534  | Get user notifications   |
| PUT    | `/api/notifications/{notification_id}/read`   | 1582  | Mark one as read         |
| PUT    | `/api/notifications/user/{user_id}/read-all`  | 1598  | Mark all as read         |

**Website Analysis**

| Method | Path                                 | Line  | Purpose                          |
|--------|--------------------------------------|-------|----------------------------------|
| POST   | `/api/website/analyze`               | 1622  | Analyze website                  |
| POST   | `/api/website/screenshot`            | 1711  | Capture screenshot (POST)        |
| GET    | `/api/website/screenshot`            | 2749  | Capture screenshot (GET)         |
| POST   | `/api/website/favicon`               | 1765  | Get favicon (POST)               |
| GET    | `/api/website/favicon`               | 2759  | Get favicon (GET)                |
| POST   | `/api/website/analysis`              | 2769  | Website analysis (alternate)     |
| POST   | `/api/website/analysis_complete`     | 3358  | Complete analysis pipeline       |
| POST   | `/api/website/analyze-background`    | 2080  | Background analysis              |
| GET    | `/api/website/task/{task_id}`        | 2119  | Check background task status     |
| POST   | `/api/scrape`                        | 8943  | Generic web scrape               |

**GoHighLevel (GHL)**

| Method | Path                                                    | Line  | Purpose                          |
|--------|---------------------------------------------------------|-------|----------------------------------|
| POST   | `/api/ghl/create-subaccount`                            | 4021  | Create GHL sub-account           |
| POST   | `/api/ghl/create-user`                                  | 4218  | Create GHL user                  |
| POST   | `/api/ghl/create-subaccount-and-user`                   | 4696  | Combined creation                |
| POST   | `/api/ghl/create-subaccount-and-user-registration`      | 5577  | Combined creation (registration) |
| POST   | `/api/ghl/trigger-pit-automation/{ghl_record_id}`       | 5728  | Trigger PIT token automation     |
| POST   | `/api/ghl/retry-automation`                             | 5784  | Retry failed automation          |
| POST   | `/api/ghl/refresh-tokens/{firm_user_id}`                | 5960  | Refresh GHL tokens               |
| GET    | `/api/ghl/user/{user_id}/integrations`                  | 6033  | Get user integrations            |
| GET    | `/api/ghl/status/{ghl_record_id}`                       | 6057  | Check GHL status                 |
| POST   | `/api/ghl/get-location-id`                              | 6708  | Get location ID                  |
| POST   | `/api/ghl/run-complete-automation`                      | 7695  | Full automation pipeline         |
| POST   | `/api/ghl/refresh-firebase-token`                       | 7750  | Refresh Firebase token           |

**Facebook**

| Method | Path                                                    | Line  | Purpose                          |
|--------|---------------------------------------------------------|-------|----------------------------------|
| POST   | `/api/facebook/extract-oauth-params`                    | 6222  | Extract OAuth params             |
| POST   | `/api/facebook/start-oauth-with-interception`           | 6273  | Start OAuth interception         |
| GET    | `/api/facebook/oauth-interception-status/{session_id}`  | 6439  | Check OAuth status               |
| GET    | `/api/facebook/oauth-health`                            | 6468  | OAuth health check               |
| POST   | `/api/facebook/integrate`                               | 6495  | Integrate Facebook               |
| GET    | `/api/facebook/integration-status`                      | 6552  | Get integration status           |
| POST   | `/api/facebook/integration-status/reset/{location_id}`  | 6557  | Reset integration status         |
| GET    | `/api/facebook/integration-status/{location_id}`        | 6565  | Check location integration       |
| POST   | `/api/facebook/check-accounts-after-oauth`              | 6583  | Check accounts post-OAuth        |
| POST   | `/api/facebook/check-integration-status`                | 6752  | Check integration (POST)         |
| POST   | `/api/facebook/save-selected-pages`                     | 6824  | Save selected pages              |
| POST   | `/api/facebook/get-pages-from-integration`              | 6907  | Get pages from integration       |
| POST   | `/api/facebook/connect-selected-pages`                  | 7061  | Connect selected pages           |
| GET    | `/api/facebook/get-connection-status`                   | 7193  | Get connection status            |
| POST   | `/api/facebook/retry-token-capture`                     | 7556  | Retry token capture              |

**Business Profile**

| Method | Path                                     | Line  | Purpose                 |
|--------|------------------------------------------|-------|-------------------------|
| POST   | `/api/business/upload-logo`              | 7262  | Upload business logo    |
| POST   | `/api/business/save-profile`             | 7363  | Save business profile   |
| GET    | `/api/business/profile/{firm_user_id}`   | 7439  | Get business profile    |
| POST   | `/api/agents/notify-enablement`          | 9237  | Agent enablement notify |

**File Processing**

| Method | Path                                    | Line  | Purpose                       |
|--------|-----------------------------------------|-------|-------------------------------|
| POST   | `/api/file/process`                     | 7889  | Process uploaded file         |
| POST   | `/api/file/extract-text`                | 7975  | Extract text from file        |
| GET    | `/api/file/status/{file_id}`            | 8060  | Check file processing status  |
| GET    | `/api/files/user/{firm_user_id}`        | 8092  | List user files               |
| GET    | `/api/file/status-stream/{file_id}`     | 8315  | SSE stream for file status    |

**Knowledge Base (in main.py)**

| Method | Path                                   | Line  | Purpose                          |
|--------|----------------------------------------|-------|----------------------------------|
| POST   | `/api/knowledge-base/text`             | 8116  | Add text to knowledge base       |
| POST   | `/api/knowledge-base/file`             | 8177  | Add file to knowledge base       |
| GET    | `/api/knowledge-base/{agent_id}`       | 8726  | Get agent knowledge base         |

**Internal/Utility**

| Method | Path                          | Line  | Purpose                   |
|--------|-------------------------------|-------|---------------------------|
| POST   | `/api/supabase/query`         | 9001  | Direct Supabase query proxy|

### 4.2 Mounted Router Endpoints (not in main.py)

These are in separate files and mounted via `app.include_router()`:

- `routes/knowledge_base.py` -- prefix `/api/knowledge-base` (CRUD + semantic search)
- `routes/ghl_media.py` -- GHL media endpoints
- `routes/templated_io.py` -- Templated.io template management
- `GHL_Marketing/social_facebook.py` -- prefix `/api/social/facebook`
- `GHL_Marketing/social_instagram.py` -- prefix `/api/social/instagram`
- `mcp/server.py` -- prefix `/api/v1/mcp`

---

## 5. Core Business Logic

### 5.1 ConversationalHandler (lines 44-515)

The central message routing class. Defined as a class instantiated once at module level.

**Flow:**
1. Receive user message via WebSocket or HTTP
2. Check in-memory cache (5-minute TTL) for duplicate requests
3. Deduplicate messages (10-second window check against `chat_history` table)
4. Forward to N8N webhook at `N8N_MAIN` URL
5. Save user message and agent response to `chat_history` in Supabase
6. Return response to caller

**Key properties:**
- `_cache`: dict-based in-memory cache (not shared across workers)
- `_cache_ttl`: 300 seconds (5 minutes)
- Duplicate detection: queries `chat_history` for same message within last 10 seconds

**AI WARNING:** The cache is per-process. With 2 Gunicorn workers, each worker has its own cache. This means cache misses can occur if requests hit different workers.

### 5.2 Knowledge Base Pipeline

**Two databases involved:**
- **Supabase**: stores file metadata and raw content
- **Neon PostgreSQL**: stores vector embeddings in `user_vector_knowledge_base` table

**Flow:**
1. User uploads text or file
2. Content extracted (PDF via pdfplumber, DOCX via python-docx, TXT direct)
3. Content chunked into 1536-character segments
4. Each chunk embedded via OpenRouter API (`text-embedding-3-small`, 1536 dimensions)
5. Embeddings stored in Neon with pgvector
6. Semantic search queries embed the query, then use cosine similarity in pgvector

**AI WARNING:** The chunk size of 1536 characters is a coincidence with the 1536-dimension embeddings. They are unrelated values. Do not "fix" one thinking it needs to match the other.

### 5.3 GHL Account Automation

**Flow:**
1. Backend creates GHL sub-account via API
2. Creates GHL user within that sub-account
3. Triggers PIT (Private Integration Token) automation
4. PIT automation runs on a separate Heroku dyno (`AUTOMATION_USER1_SERVICE_URL`)
5. Uses Playwright browser automation to capture tokens

**AI WARNING:** The PIT automation is fragile. It depends on GHL's web UI remaining stable. UI changes on GHL's side will break it. The separate automation service URL is `https://backgroundautomationuser1-1644057ede7b.herokuapp.com`.

### 5.4 Facebook OAuth Interception

Uses `facebook_oauth_interceptor.py` to intercept OAuth tokens during the Facebook integration flow. This is an unusual pattern where Playwright monitors network requests during an OAuth flow to capture tokens that are not normally exposed via API.

**AI WARNING:** This is inherently fragile and error-prone. It depends on Facebook's OAuth flow not changing.

### 5.5 Website Analysis

**Flow:**
1. Playwright captures screenshot and favicon
2. BeautifulSoup scrapes page content
3. Perplexity AI analyzes scraped content for business insights
4. Color extraction from website for branding

**Note:** Both sync (`/api/website/analyze`) and background (`/api/website/analyze-background`) versions exist. Background version returns a `task_id` for polling via `/api/website/task/{task_id}`.

### 5.6 File Processing

**Flow:**
1. File uploaded via `/api/file/process` or `/api/file/extract-text`
2. `FileProcessingService` handles extraction by file type
3. `background_text_processor` runs extraction asynchronously
4. Progress streamed via SSE at `/api/file/status-stream/{file_id}`
5. Status tracked in `file_processing_status` dict (in-memory, per-worker)

### 5.7 Real-time Chat (WebSocket)

- Endpoint: `/ws/{user_id}/{session_id}`
- Implements ping/pong keepalive
- Message queue for handling concurrent sends
- Routes messages through `ConversationalHandler` to N8N
- Active connections tracked in `active_connections` dict with thread lock

---

## 6. Databases

### 6.1 Supabase PostgreSQL

**Connection:** Via `supabase-py` SDK using `SUPABASE_URL` + `SUPABASE_SERVICE_KEY`

**Key tables:**
- `chat_history` -- all user/agent messages with session tracking
- `users` / `profiles` -- user accounts and profile data
- `ghl_credentials` -- GoHighLevel tokens and sub-account info
- `business_profiles` -- business settings, logos, branding
- `notifications` -- user notification queue
- `facebook_pages` -- connected Facebook pages
- `agents` -- agent configuration and enablement per user
- Various GHL-related tables for automation state

### 6.2 Neon PostgreSQL (Vector Search)

**Connection:** Direct via `asyncpg` using `NEON_DB_*` environment variables

**Key table:** `user_vector_knowledge_base`
- `id` (UUID)
- `user_id` (text)
- `agent_id` (text)
- `content` (text) -- the text chunk
- `embedding` (vector(1536)) -- pgvector column
- `metadata` (jsonb) -- source file info, chunk index
- `created_at` (timestamp)

**Search function:** Uses cosine similarity (`<=>` operator) in pgvector.

---

## 7. Environment Variables

### Required

| Variable                   | Purpose                                          |
|----------------------------|--------------------------------------------------|
| `SUPABASE_URL`             | Supabase project URL                             |
| `SUPABASE_SERVICE_KEY`     | Supabase service role key (admin access)         |
| `OPENROUTER_API_KEY`       | OpenRouter API for embeddings                    |
| `PERPLEXITY_API_KEY`       | Perplexity AI for web search/analysis            |
| `NEON_DB_HOST`             | Neon PostgreSQL host                             |
| `NEON_DB_USER`             | Neon database user                               |
| `NEON_DB_PASSWORD`         | Neon database password                           |

### Optional / With Defaults

| Variable                       | Default                                               | Purpose                         |
|--------------------------------|-------------------------------------------------------|---------------------------------|
| `N8N_MAIN`                     | `https://n8n.theaiteam.uk/webhook/c2fcbad6-...`       | Primary N8N webhook URL         |
| `N8N_MAIN_TEST`                | (none)                                                | Test N8N webhook                |
| `NEON_DB_PORT`                 | `5432`                                                | Neon PostgreSQL port            |
| `NEON_DB_NAME`                 | `neondb`                                              | Neon database name              |
| `GHL_COMPANY_ID`               | `lp2p1q27DrdGta1qGDJd`                               | GoHighLevel company ID          |
| `GHL_AGENCY_TOKEN`             | `pit-e3d8d384-00cb-4744-8213-b1ab06ae71fe`            | GHL agency-level PIT token      |
| `GHL_SNAPSHOT_ID`              | `bInwX5BtZM6oEepAsUwo`                               | GHL snapshot for sub-accounts   |
| `GHL_WEBHOOK_SECRET`           | (none)                                                | GHL webhook verification        |
| `AUTOMATION_USER1_SERVICE_URL` | `https://backgroundautomationuser1-...herokuapp.com`  | External automation dyno        |
| `WEB_CONCURRENCY`              | `2`                                                   | Gunicorn worker count           |
| `ENVIRONMENT`                  | `production`                                          | App environment flag            |
| `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD` | `1` (set in app.json)                            | Skip pip-time browser download  |
| `OPENAI_API_KEY`               | (commented out in code)                               | OpenAI key (currently unused)   |

---

## 8. Deployment

### 8.1 Heroku Configuration

**Buildpacks (in order):**
1. `heroku-buildpack-apt` -- installs system-level packages (Chromium dependencies)
2. `heroku/python` -- installs Python and pip dependencies

**Addons:**
- `rediscloud:30` -- Redis instance (declared in app.json, availability depends on plan)

**Procfile sequence:**
```
chmod +x install_chromium.sh
./install_chromium.sh          # Installs Playwright Chromium to /app/browsers
WEB_CONCURRENCY=2 gunicorn main:app --config gunicorn_config.py --preload
```

### 8.2 Gunicorn Configuration (`gunicorn_config.py`)

```python
workers = 2                    # Heroku 512MB limit -- do not increase
worker_class = 'uvicorn.workers.UvicornWorker'
timeout = 29                   # Under Heroku 30s hard limit
graceful_timeout = 29
keepalive = 5
preload_app = True             # Load app once, fork to workers (saves memory)
worker_connections = 500
max_requests = 200             # Restart workers after 200 requests (leak prevention)
max_requests_jitter = 50       # Randomize restart to avoid all-at-once
threads = 1                    # Single thread per worker
```

### 8.3 Slug Size Management

The `.slugignore` file excludes 70+ patterns including:
- `Tools/` directory (entirely redundant -- main.py imports from root-level files)
- All `Database/` SQL files
- All `test_*.py` files
- All `*.md` documentation
- Multiple alternate `requirements*.txt` files
- `static/screenshots/*` and `static/favicons/*`
- Python cache, virtual environments, build artifacts

### 8.4 Deployment Checklist

1. Ensure `runtime.txt` says `python-3.12` (not 3.13)
2. Verify `requirements.txt` is the canonical deps file
3. Confirm `Aptfile` has Chromium dependencies
4. Push to Heroku git remote
5. Monitor build logs for Playwright Chromium install
6. Verify health check at `GET /health`

---

## 9. CORS Configuration

CORS middleware is added at the **very end** of main.py (line 9328), after all routes:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**AI WARNING:** This is a wildcard CORS policy. It allows any origin. This is acceptable during development but should be restricted in production. However, changing it may break the frontend, mobile apps, or N8N integrations. Do not change without testing all consumers.

---

## 10. In-Memory State

The following state is held in Python process memory (NOT persisted, NOT shared across workers):

| Variable                     | Type                        | Purpose                               | Thread-safe? |
|------------------------------|-----------------------------|---------------------------------------|--------------|
| `active_connections`         | `Dict[str, WebSocket]`      | Active WebSocket connections          | Yes (lock)   |
| `streaming_sessions`         | `Dict[str, Dict]`           | Active SSE streaming sessions         | Yes (lock)   |
| `request_cache`              | `Dict[str, float]`          | Request deduplication timestamps      | Yes (lock)   |
| `file_processing_status`     | `Dict[str, Dict]`           | File processing progress              | Yes (lock)   |
| `background_results`         | `Dict`                      | Background task results               | No           |
| `running_tasks`              | `Dict[str, Dict]`           | Running background tasks              | No           |
| `ConversationalHandler._cache` | `Dict`                    | Response cache (5-min TTL)            | No           |

**AI WARNING:** With 2 Gunicorn workers, each worker has its own copy of all in-memory state. A WebSocket connected to worker 1 cannot see file processing status updates happening in worker 2. This is a known limitation.

---

## 11. Initialization Sequence

When the application starts, the following happens in order:

1. `load_dotenv()` loads environment variables (line 517)
2. `FastAPI()` app created (line 519)
3. Logging configured (lines 522-528)
4. Thread locks created for concurrent data structures (lines 530-542)
5. `create_supabase_client()` creates global Supabase client (line 577)
6. `ConversationalHandler` instantiated with Supabase client (line 581)
7. `FileProcessingService` instantiated (line 587)
8. `initialize_background_processor()` called (line 588)
9. All `@app.route` decorators registered (lines 779-9237)
10. External routers mounted via `app.include_router()` (lines 9092-9183)
11. CORS middleware added (line 9328)
12. If `__main__`, Uvicorn starts directly (line 9337)

**AI WARNING:** Because `preload_app = True` in Gunicorn config, the app is loaded once in the master process and then forked. The Supabase client and handler objects are created before forking, then copied to each worker. This is generally fine but means each worker has independent state.

---

## 12. Tribal Knowledge and Gotchas

This section is critical reading. These are lessons learned the hard way.

### 12.1 The Monolith Problem

`main.py` is 9,338 lines. It contains 74 endpoint decorators, ~20 Pydantic models, the ConversationalHandler class, and hundreds of helper functions. This is the single largest piece of technical debt in the project.

**Why it matters:**
- Merge conflicts are frequent
- Finding code requires searching by line number
- IDE performance degrades
- Refactoring is risky because everything is coupled

**What NOT to do:**
- Do not add more endpoints to main.py if you can avoid it. Put new routes in `routes/`.
- Do not attempt a full refactor in one PR. Extract one domain at a time.

**What to do:**
- New features should go in `routes/` as separate router modules.
- Follow the pattern used by `routes/knowledge_base.py` or `routes/templated_io.py`.

### 12.2 Python Version Lock

Python 3.12 is REQUIRED. Python 3.13 has known incompatibilities with one or more of the dependencies. The `runtime.txt` file pins this. Do not upgrade without testing the full dependency tree.

### 12.3 Heroku Memory Constraints

The dyno has 512MB of RAM. With `preload_app = True` and 2 workers, memory is tight. Each worker also loads Playwright's Chromium when needed.

**Rules:**
- Never increase `WEB_CONCURRENCY` above 2
- Never remove `max_requests = 200` (it prevents memory leaks from accumulating)
- Watch for memory-hungry operations (large PDF processing, multiple simultaneous Playwright sessions)

### 12.4 The 29-Second Timeout

Heroku kills any request that takes longer than 30 seconds. Gunicorn is configured for 29 seconds to stay under this. Long-running operations MUST use background tasks:
- Website analysis has a background variant (`/api/website/analyze-background`)
- File processing uses background tasks with SSE status streaming
- GHL automation delegates to an external automation service

If you add a new endpoint that might take more than ~25 seconds, it MUST be async with a background task pattern.

### 12.5 Tools/ Directory is Dead Code (in Production)

The `.slugignore` excludes the entire `Tools/` directory. `main.py` imports from root-level files (`Website/`, `invitation_handler.py`, etc.), NOT from `Tools/`. The `Tools/` directory contains GHL API wrappers that appear to be an older or reference implementation.

Do not add production code to `Tools/`. It will not be deployed.

### 12.6 Duplicate Endpoints

Several endpoints exist in multiple forms with slight variations:
- `/api/website/screenshot` exists as both GET (line 2749) and POST (line 1711)
- `/api/website/favicon` exists as both GET (line 2759) and POST (line 1765)
- `/api/website/analyze` vs `/api/website/analysis` vs `/api/website/analysis_complete`
- `/api/ghl/create-subaccount-and-user` vs `/api/ghl/create-subaccount-and-user-registration`
- Knowledge base routes exist BOTH in main.py AND in `routes/knowledge_base.py`

This is technical debt from iterative development. Before adding a new endpoint, search for existing ones that do the same thing.

### 12.7 Knowledge Base Dual-Database Pattern

Knowledge base data lives in TWO places:
1. **Supabase**: file metadata, raw text, user-facing CRUD
2. **Neon (pgvector)**: vector embeddings for semantic search

Both must be kept in sync. Deleting a knowledge base entry must clean up both databases. The `routes/knowledge_base.py` router handles Neon directly via `asyncpg`, while main.py KB routes may use Supabase. Be aware of which database you are talking to.

### 12.8 Embedding Configuration

- Model: `text-embedding-3-small` via OpenRouter
- Dimensions: 1536
- Chunk size: 1536 characters (coincidental match with dimension count -- not related)
- Similarity metric: cosine distance (`<=>` operator in pgvector)

### 12.9 ConversationalHandler Cache Behavior

- Cache is a plain Python dict (not Redis, not shared)
- TTL: 5 minutes
- Duplicate prevention: 10-second window query against Supabase `chat_history`
- Each of the 2 Gunicorn workers has its own cache
- Redis is available (app.json declares `rediscloud:30`) but is NOT currently used for caching

### 12.10 GHL Automation External Service

PIT token automation and some retry flows delegate to an external Heroku app:
```
AUTOMATION_USER1_SERVICE_URL = https://backgroundautomationuser1-1644057ede7b.herokuapp.com
```
This is a separate deployment with its own Playwright instance for browser automation against the GHL web UI. If that service is down, PIT automation fails silently or with an HTTP error.

### 12.11 Facebook OAuth Interception

The Facebook OAuth flow uses a Playwright-based interception pattern (`facebook_oauth_interceptor.py`). This is NOT a standard OAuth code exchange. It intercepts network traffic during the browser-based OAuth flow to capture tokens. This is inherently fragile and will break if Facebook changes their OAuth UI or network patterns.

### 12.12 No Test Framework

There is no pytest, unittest, or any formal testing configuration. The `test_*.py` files in the project root are ad-hoc scripts meant to be run manually. They are excluded from the Heroku slug via `.slugignore`.

### 12.13 CORS at End of File

The CORS middleware is added at line 9328, after all route definitions. In FastAPI, middleware order matters. CORS middleware should generally be added before routes, but in this codebase it works because FastAPI handles this internally. However, be aware that if you add middleware of your own, insertion order relative to CORS matters.

### 12.14 OpenAI SDK Installed but Unused

The `openai==1.58.1` package is in requirements.txt, and the import is commented out in main.py (line 25: `# from openai import OpenAI`). The actual embedding calls go through OpenRouter via `httpx`. Do not assume OpenAI SDK is available for use without uncommenting and configuring it.

---

## 13. Data Flow Diagrams

### 13.1 Chat Message Flow

```
User (Frontend)
    |
    v
WebSocket /ws/{user_id}/{session_id}
    |
    v
ConversationalHandler.handle_message()
    |
    +-- Check cache (in-memory, 5min TTL)
    |
    +-- Deduplicate (Supabase chat_history, 10s window)
    |
    v
POST to N8N webhook (N8N_MAIN)
    |
    v
N8N processes via AI Agent workflow
    |
    v
Response returned to ConversationalHandler
    |
    +-- Save to chat_history (Supabase)
    +-- Cache response (in-memory)
    |
    v
Response sent via WebSocket to User
```

### 13.2 Knowledge Base Upload Flow

```
User uploads file
    |
    v
POST /api/knowledge-base/file  OR  POST /api/knowledge-base/text
    |
    v
Extract text (pdfplumber / python-docx / raw)
    |
    v
Chunk into 1536-char segments
    |
    v
For each chunk:
    +-- POST to OpenRouter (text-embedding-3-small)
    +-- Get 1536-dim embedding vector
    +-- INSERT into Neon user_vector_knowledge_base
    |
    v
Store metadata in Supabase
    |
    v
Return success to user
```

### 13.3 GHL Account Setup Flow

```
User triggers setup
    |
    v
POST /api/ghl/create-subaccount-and-user
    |
    +-- Create sub-account via GHL API
    +-- Wait for location availability (polling, up to 10 retries)
    +-- Create user in sub-account via GHL API
    +-- Store credentials in Supabase
    |
    v
POST /api/ghl/trigger-pit-automation/{id}
    |
    v
Delegate to AUTOMATION_USER1_SERVICE_URL
    |
    v
External service uses Playwright to:
    +-- Login to GHL web UI
    +-- Navigate to PIT settings
    +-- Capture PIT token
    +-- Return token
    |
    v
Store PIT token in Supabase
```

---

## 14. Recommended Refactoring Path

For AI agents or developers tasked with improving this codebase, here is the recommended incremental approach:

### Phase 1: Stop Making It Worse
- All new endpoints go in `routes/` as separate router modules
- Follow the pattern in `routes/knowledge_base.py`
- Do not add lines to main.py

### Phase 2: Extract by Domain
Extract these domains into their own route modules, one at a time:
1. `routes/auth.py` -- all `/api/auth/*` endpoints (~8 endpoints, lines 2127-2749)
2. `routes/website.py` -- all `/api/website/*` endpoints (~10 endpoints, lines 1622-3358)
3. `routes/ghl.py` -- all `/api/ghl/*` endpoints (~12 endpoints, lines 4021-7889)
4. `routes/facebook.py` -- all `/api/facebook/*` endpoints (~15 endpoints, lines 6222-7641)
5. `routes/chat.py` -- WebSocket + chat history (~3 endpoints)
6. `routes/files.py` -- all `/api/file/*` endpoints (~5 endpoints)
7. `routes/business.py` -- all `/api/business/*` endpoints (~3 endpoints)
8. `routes/notifications.py` -- all `/api/notifications/*` endpoints (~3 endpoints)

### Phase 3: Extract Shared Logic
- Move `ConversationalHandler` to its own module
- Move Pydantic models to a `models/` directory
- Move helper functions (URL normalization, image URL extraction) to `utils/`

### Phase 4: Add Testing
- Set up pytest with fixtures
- Start with integration tests for critical flows (chat, KB upload, auth)
- Add the test configuration to CI

### Phase 5: Use Redis
- Replace in-memory caches with Redis (already provisioned via `rediscloud:30`)
- This fixes the multi-worker cache inconsistency problem

---

## 15. For AI Coding Agents

### Before making changes:

1. **Read main.py in sections.** Do not try to read the entire 9,338-line file at once. Use line ranges.
2. **Search for existing endpoints** before creating new ones. Duplicates already exist; do not add more.
3. **Check `.slugignore`** before adding files. If your file matches a pattern there, it will not be deployed.
4. **New routes go in `routes/`**, not in main.py. Mount them with `app.include_router()` in the router-mounting section near line 9092.
5. **Test locally with `uvicorn main:app`** before assuming Heroku deployment will work.

### Common pitfalls:

- Adding code to `Tools/` -- it is excluded from deployment.
- Using Python 3.13 features -- the runtime is locked to 3.12.
- Creating long-running synchronous endpoints -- you have 29 seconds maximum.
- Assuming in-memory state is shared across requests -- it is not (2 workers).
- Adding large dependencies to `requirements.txt` -- the 512MB memory limit is real.
- Modifying CORS without testing all consumers.

### When adding a new endpoint:

```python
# In routes/my_new_feature.py
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/api/my-feature", tags=["my_feature"])

class MyRequest(BaseModel):
    field: str

@router.post("/action")
async def my_action(request: MyRequest):
    # implementation
    return {"success": True}
```

Then in main.py, near line 9092:
```python
try:
    from routes.my_new_feature import router as my_feature_router
    app.include_router(my_feature_router)
    logger.info("My Feature routes loaded successfully")
except ImportError as e:
    logger.warning(f"My Feature routes not available: {e}")
```

---

*This document was generated from direct analysis of the codebase at `/Users/sethward/GIT/Squidgy/squidgy_updated_backend/` on 2026-02-22.*
