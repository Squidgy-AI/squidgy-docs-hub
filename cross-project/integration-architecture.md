# Squidgy Platform: Cross-Project Integration Architecture

**Audience:** Human developers onboarding to the project AND AI coding agents working in the codebase.
**Last updated:** 2026-02-22

---

## Table of Contents

1. [Platform Overview](#platform-overview)
2. [The Three Projects](#the-three-projects)
3. [Shared Infrastructure](#shared-infrastructure)
4. [Integration Flows](#integration-flows)
   - [Flow 1: Agent Creation Pipeline](#flow-1-agent-creation-pipeline)
   - [Flow 2: Chat Message Flow](#flow-2-chat-message-flow)
   - [Flow 3: Agent Enablement Flow](#flow-3-agent-enablement-flow)
   - [Flow 4: Website Analysis Flow](#flow-4-website-analysis-flow)
   - [Flow 5: Knowledge Base Flow](#flow-5-knowledge-base-flow)
   - [Flow 6: GoHighLevel Integration Flow](#flow-6-gohighlevel-integration-flow)
   - [Flow 7: N8N Workflow Backup Flow](#flow-7-n8n-workflow-backup-flow)
5. [Environment Variables](#environment-variables)
6. [Cross-Project Gotchas](#cross-project-gotchas)
7. [Debugging Checklist](#debugging-checklist)

---

## Platform Overview

Squidgy is an AI agent platform where users interact with specialized AI agents (brand advisors, personal assistants, etc.) through a chat interface. The platform is split across three codebases that share infrastructure and generate artifacts consumed by each other.

```
+------------------------------------------------------------------+
|                        SQUIDGY PLATFORM                          |
|                                                                  |
|  +---------------------+    +---------------------+             |
|  | squidgy_updated_ui  |    | squidgy_updated_    |             |
|  | (Frontend)          |    | backend (Python)    |             |
|  |                     |    |                     |             |
|  | React SPA +         |    | FastAPI on Heroku   |             |
|  | Express server      |    | 75+ API endpoints   |             |
|  +--------+--+---------+    +---+-------+---------+             |
|           |  |                  |       |                        |
|           |  |   +--------------+       |                        |
|           |  |   |                      |                        |
|  +--------v--v---v--+    +-------------v---------+              |
|  |     Supabase      |    |       Neon DB         |              |
|  | (Auth, Storage,   |    | (pgvector embeddings) |              |
|  |  Shared Tables)   |    +----------------------+              |
|  +-------------------+                                           |
|           |                                                      |
|  +--------v----------+    +----------------------+              |
|  |       N8N          |    | squidgy_agent_       |              |
|  | (Workflow Engine)  |<---| builder (CLI)        |              |
|  | Agent conversations|    | Generates configs,   |              |
|  | + scheduled tasks  |    | workflows, deploys   |              |
|  +--------------------+    +----------------------+              |
+------------------------------------------------------------------+
```

---

## The Three Projects

### 1. squidgy_updated_ui (Frontend)

**What it is:** A React single-page app served by an Express backend-for-frontend (BFF) server.

**Key responsibilities:**
- Chat interface for all agent conversations
- Agent management sidebar (enable/disable agents)
- Onboarding wizard (website analysis, branding)
- File upload for knowledge base
- Proxy routes to the Python backend (website analysis, GHL)

**Key paths for AI coders:**
- React components: `client/components/`
- API utilities: `client/lib/api.ts`
- Agent configs (YAML): `agents/configs/*.yaml`
- Knowledge base content: `knowledge_base/agents/{agent_id}/`
- N8N workflow backups: `n8n/`
- Build script: `scripts/build-agents.js`
- Express server routes: `server/`

**Important distinction:** The Express server in this project is NOT the Python backend. It is a lightweight BFF that handles:
- Static file serving
- Proxying requests to the Python backend
- GHL integration routes
- File storage routes

### 2. squidgy_updated_backend (Backend)

**What it is:** A Python FastAPI application deployed on Heroku. This is where the heavy computation happens.

**Key responsibilities:**
- Website scraping and analysis (Playwright + Perplexity AI)
- Knowledge base processing (text extraction, chunking, embedding)
- Semantic search over knowledge base (Neon pgvector)
- GoHighLevel sub-account creation and automation
- N8N agent KB query endpoint (`POST /n8n/agent/query`)

**Key paths for AI coders:**
- API routes: `routes/`
- Database migrations: `migrations/`
- Utility tools: `Tools/`

### 3. squidgy_agent_builder (CLI Tool)

**What it is:** A TypeScript CLI tool that generates the full agent pipeline. It produces artifacts that live in the frontend repo.

**Key responsibilities:**
- Interactive agent creation wizard (9 steps)
- Generates YAML config files
- Generates N8N workflow JSON
- Updates Pia's system prompt (3 tables)
- Deploys workflows to N8N via API
- Runs `build-agents.js` to sync everything

**Key paths for AI coders:**
- CLI entry: `scripts/create-agent-interactive.js`
- Core service: `src/services/agentBuilderService.ts`
- Templates: `templates/`
- Example agents: `examples/`
- N8N pattern docs: `docs/n8n/`

---

## Shared Infrastructure

### Supabase (Primary Database + Auth + Storage)

Both frontend and backend connect to the same Supabase project. They share tables but use different SDKs.

```
+-----------------------+     +-----------------------+
| Frontend              |     | Backend               |
| @supabase/supabase-js |     | Python supabase SDK   |
| PKCE auth flow        |     | Service role key      |
+-----------+-----------+     +-----------+-----------+
            |                             |
            +----------+  +--------------+
                       |  |
                +------v--v------+
                |    Supabase    |
                |                |
                | Tables:        |
                | - profiles     |
                | - agents       |
                | - personal_    |
                |   assistant_   |
                |   config       |
                | - chat_history |
                | - brands       |
                | - ghl_         |
                |   subaccounts  |
                | - business_    |
                |   settings     |
                | - website_     |
                |   analysis     |
                |                |
                | Storage:       |
                | - KB file      |
                |   uploads      |
                +----------------+
```

### Neon (Vector Database)

A separate PostgreSQL database with pgvector extension. Used exclusively for knowledge base embeddings. Only the Python backend reads/writes to Neon.

```
Frontend uploads file --> Supabase Storage (file blob)
                              |
                    Python backend downloads file
                              |
                    Extract text --> Chunk (1536 chars)
                              |
                    OpenRouter text-embedding-3-small
                              |
                    Store chunks + vectors --> Neon pgvector
```

### N8N (Workflow Engine)

Self-hosted at `https://n8n.theaiteam.uk`. Serves as the AI conversation engine for all agents.

**Three systems interact with N8N:**

| System | How it uses N8N |
|--------|----------------|
| Agent Builder | Deploys workflow JSON via N8N REST API |
| Frontend | Calls webhook endpoints for agent conversations |
| Backend | Receives KB query requests from N8N workflows |

**API key location:** `/Users/sethward/GIT/Squidgy/.n8n-api-key`

---

## Integration Flows

### Flow 1: Agent Creation Pipeline

This is the most cross-cutting flow. The agent builder CLI generates artifacts in the frontend repo, deploys to N8N, and syncs to Supabase.

```
 Developer runs CLI
        |
        v
+------------------+
| Agent Builder CLI |
| (squidgy_agent_  |
|  builder)        |
+--------+---------+
         |
         | Step 1: Generate YAML
         +------> squidgy_updated_ui/agents/configs/{id}.yaml
         |
         | Step 2: Generate N8N Workflow
         +------> squidgy_updated_ui/n8n/{id}_workflow.json
         |
         | Step 3: Update Pia System Prompt
         +------> squidgy_updated_ui/knowledge_base/agents/
         |           personal_assistant/system_prompt.md
         |        (Updates 3 tables: relevance rules, agent IDs, routing)
         |
         | Step 4: Deploy to N8N
         +------> POST https://n8n.theaiteam.uk/api/v1/workflows
         |
         | Step 5: Run build-agents.js
         +------> squidgy_updated_ui/scripts/build-agents.js
                    |
                    +---> Reads all YAML configs
                    +---> Generates agents.ts (typed agent registry)
                    +---> Syncs to Supabase:
                            - agents table
                            - personal_assistant_config table
```

**Where things break:**
- If you skip step 5 (build-agents.js), the frontend will not see the new agent.
- If you skip step 3 (Pia prompt update), Pia will not route users to the new agent.
- N8N API deployment (step 4) cannot set credentials or activate the workflow. You must do this in the N8N UI manually.
- After deployment, you must: set credentials in N8N UI, activate the workflow, and restart Pia's workflow if the system prompt changed.

**For AI coders:** When told to "add a new agent," you need to touch files in BOTH `squidgy_agent_builder` (templates/service logic) AND `squidgy_updated_ui` (where output files land). The agent builder is the tool; the frontend repo is where its output lives.

---

### Flow 2: Chat Message Flow

This is the primary runtime flow. Messages go directly from the frontend to N8N, bypassing the Python backend entirely.

```
+------------------+         +-------------------+         +------------------+
|   React UI       |         |      N8N          |         |   Supabase       |
|   ChatInterface  | POST    |   Agent Workflow  |         |                  |
|   .tsx           +-------->+                   +-------->+  getAll user     |
|                  |         |  1. Webhook        |  query  |  data (brands,  |
|                  |         |  2. Supabase getAll |<-------+  settings, etc) |
|                  |         |  3. Code node      |         |                  |
|                  |         |     (merge data)   |         +------------------+
|                  |         |  4. AI Agent       |
|                  |   JSON  |     (OpenRouter/   |
|                  |<--------+      Claude)       |
|                  |         |  5. Output Parser  |
|                  |         |  6. If Ready/Wait  |
|                  |         |  7. Respond node   |
+--------+---------+         +-------------------+
         |
         | After receiving response:
         |
         +---> Update chat UI
         +---> conversationStateService: persist state locally
         +---> chatHistoryService: save to Supabase chat_history table
```

**Request payload (frontend sends):**
```json
{
  "body": {
    "user_id": "uuid",
    "user_mssg": "Tell me about branding",
    "session_id": "uuid",
    "request_id": "uuid",
    "state": { ... }
  }
}
```

**Response payload (N8N returns):**
```json
{
  "response": "Here's what I think about branding...",
  "Status": "Ready",
  "state": { "current_step": "analysis", ... }
}
```

**Where things break:**
- The field is `Status` (capital S) in the output parser schema, not `status`.
- The field is `state` (lowercase) everywhere. Not `conversation_state`.
- Frontend reads state at `$input.first().json.body.state` in N8N Code nodes.
- If the N8N Respond node never fires (workflow error), the response body will be 0 bytes. Test with `curl` and check body size.
- Supabase nodes in N8N must use `getAll` (not `get`) with `condition: "eq"` and `alwaysOutputData: true`. Using `get` crashes on no matching row.

**For AI coders:** Chat messages do NOT go through the Python backend. If you are debugging chat issues, look at `client/lib/api.ts` (callN8NWebhook function) and the N8N workflow, not the Python routes.

---

### Flow 3: Agent Enablement Flow

Users discover and enable agents through Pia (the Personal Assistant).

```
+----------------+       +-------------------+       +------------------+
|  User in chat  |       |   Pia (N8N        |       |    Supabase      |
|  "I need help  |------>|   Workflow)        |       |                  |
|   with         |       |                   |       |  personal_       |
|   branding"    |       |  Checks routing   |       |  assistant_      |
|                |       |  table in system   |       |  config          |
|                |<------+  prompt, suggests  |       |                  |
|                |       |  "Try Brandy"     |       |                  |
+--------+-------+       +-------------------+       +--------+---------+
         |                                                     ^
         | User clicks "Enable Brandy"                         |
         v                                                     |
+--------+-------+                                             |
| agentEnablement|   UPDATE personal_assistant_config           |
| Service        +---------------------------------------------+
|                |   SET is_enabled = true
+--------+-------+   WHERE agent_id = 'brandy'
         |
         v
+--------+-------+
| Sidebar re-    |
| renders with   |
| Brandy visible |
+----------------+
```

**Pia's system prompt** contains three tables that control agent discovery:

| Table | Location in prompt | Purpose |
|-------|-------------------|---------|
| Agent Relevance Rules (~line 215) | `\| Display Name \| Show When \|` | When Pia mentions each agent |
| Agent IDs (~line 262) | `\| Display Name \| agent_id \|` | Maps display names to IDs |
| Routing Table (~line 284) | `\| Keywords \| Route To \|` | What topics map to which agent |

**For AI coders:** When adding a new agent, you must update ALL THREE tables in `squidgy_updated_ui/knowledge_base/agents/personal_assistant/system_prompt.md`. The agent builder does this automatically, but manual agents need manual updates.

---

### Flow 4: Website Analysis Flow

Website analysis involves all three layers: frontend, Express BFF server, and Python backend.

```
+----------------+       +-------------------+       +------------------+
|  Frontend      |       |  Express BFF      |       |  Python Backend  |
|  Onboarding    | POST  |  Server           | POST  |  (Heroku)        |
|  Wizard        +------>+ /website/full-    +------>+ /api/website/    |
|                |       |  analysis          |       |  analyze         |
|                |       |                   |       |                  |
|                |       |  (Proxy route -   |       |  1. Playwright   |
|                |       |   passes through) |       |     screenshot   |
|                |       |                   |       |  2. Perplexity   |
|                |       |                   |       |     AI analysis  |
|                |       |                   |       |  3. Save to      |
|                |<------+<------------------+<------+     Supabase     |
|                |       |                   |       |  website_        |
|  Display       |       |                   |       |  analysis table  |
|  results in    |       |                   |       |                  |
|  wizard        |       |                   |       |                  |
+----------------+       +-------------------+       +------------------+
```

**Where things break:**
- The Express server is a proxy here, not the processor. If analysis fails, debug the Python backend, not the Express server.
- Playwright requires specific Heroku buildpacks to run headless Chrome.
- Background analysis variant (`/api/website/analyze-background`) is async; frontend must poll for results.

---

### Flow 5: Knowledge Base Flow

The knowledge base flow spans all three infrastructure layers: Supabase Storage, Python backend, and Neon vector DB.

```
INGESTION (User uploads document):

+----------------+       +-------------------+       +------------------+
|  Frontend      |       |  Supabase         |       |  Python Backend  |
|  File Upload   | PUT   |  Storage          | event |                  |
|  Component     +------>+  (blob store)     +------>+  1. Download     |
|                |       |                   |       |     from Storage |
|                |       |                   |       |  2. Extract text |
|                |       |                   |       |     (pdfplumber/ |
|                |       |                   |       |      python-docx)|
|                |       |                   |       |  3. Chunk text   |
|                |       |                   |       |     (1536 chars) |
|                |       |                   |       |  4. Embed via    |
|                |       |                   |       |     OpenRouter   |
|                |       |                   |       |  5. Store in     |
|                |       |                   |       |     Neon pgvector|
+----------------+       +-------------------+       +--------+---------+
                                                              |
                                                     +--------v---------+
                                                     |  Neon DB          |
                                                     |  (pgvector)       |
                                                     |  chunks +         |
                                                     |  embeddings       |
                                                     +--------+---------+
                                                              ^
RETRIEVAL (N8N agent queries KB):                             |
                                                              |
+-------------------+       +------------------+              |
|  N8N Agent        | POST  |  Python Backend  |   semantic   |
|  Workflow         +------>+ /n8n/agent/query  +-----> search |
|  (during chat)    |       |                  |              |
|                   |<------+  Returns top     |              |
|  Injects context  |       |  relevant chunks |              |
|  into AI prompt   |       +------------------+              |
+-------------------+
```

**Two databases, two purposes:**
- **Supabase Storage**: Holds the original uploaded files (PDFs, DOCX, TXT).
- **Neon pgvector**: Holds the processed text chunks and their vector embeddings for semantic search.

**For AI coders:** These are completely separate databases. Do not confuse them. Supabase connection strings and Neon connection strings are different environment variables.

---

### Flow 6: GoHighLevel Integration Flow

GHL integration creates CRM sub-accounts for users.

```
+----------------+       +-------------------+       +------------------+
|  Frontend      |       |  Express BFF      |       |  Python Backend  |
|  Onboarding    | POST  |  Server           | POST  |  (Heroku)        |
|  Wizard        +------>+ /ghl/create-     +------>+ /api/ghl/create  |
|                |       |  subaccount-     |       |                  |
|                |       |  and-user         |       |  1. Create GHL   |
|                |       |                   |       |     sub-account  |
|                |       |                   |       |  2. Playwright   |
|                |       |                   |       |     automation   |
|                |       |                   |       |     (PIT token)  |
|                |       |                   |       |  3. Store creds  |
|                |<------+<------------------+<------+     in Supabase  |
|                |       |                   |       |  ghl_subaccounts |
|  Show GHL      |       |                   |       |                  |
|  status        |       |                   |       |                  |
+----------------+       +-------------------+       +------------------+
```

---

### Flow 7: N8N Workflow Backup Flow

Automated backup of all N8N workflows to GitHub. This runs entirely within N8N.

```
+-------------------+       +------------------+       +------------------+
|  N8N Scheduled    |       |  N8N Workflow     |       |  GitHub          |
|  Trigger          +------>+  Export API       |       |  Squidgy-AI/     |
|  (MIvniarnY8AP    |       |                  |       |  N8N-Workflows   |
|   4WqF)           |       |  Export all       |       |                  |
|                   |       |  workflows as     +------>+  From_N8N_       |
|                   |       |  JSON             |       |  Scheduler/      |
|                   |       |                  |       |                  |
+-------------------+       +------------------+       +------------------+

Error handling pattern:
  GitHub Edit (update existing file)
    |
    +-- onError --> Restore Binary Data (Code node)
    |                  |
    |                  v
    |              GitHub Create (new file)
    |
    +-- onSuccess --> Done

NOTE: Binary data does NOT flow through error paths in N8N.
The Code node in the error path must re-attach the binary data
before the GitHub Create node can use it.
```

---

## Environment Variables

### Frontend (squidgy_updated_ui)

| Variable | Purpose |
|----------|---------|
| `VITE_SUPABASE_URL` | Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Supabase anonymous key (PKCE auth) |
| `VITE_N8N_WEBHOOK_URL` | Base URL for N8N webhooks (`https://n8n.theaiteam.uk/webhook`) |
| `VITE_BACKEND_URL` | Python backend URL on Heroku |

### Backend (squidgy_updated_backend)

| Variable | Purpose |
|----------|---------|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_KEY` | Supabase service role key |
| `NEON_*` | Neon DB connection (host, database, user, password) |
| `OPENROUTER_API_KEY` | For embeddings (text-embedding-3-small) and AI |
| `GHL_*` | GoHighLevel API credentials |
| `PERPLEXITY_API_KEY` | For website content analysis |

### Agent Builder (squidgy_agent_builder)

| Variable | Source | Purpose |
|----------|--------|---------|
| N8N API key | File: `/Users/sethward/GIT/Squidgy/.n8n-api-key` | Deploy workflows to N8N |

**Note:** The agent builder reads the API key from a file, not an environment variable.

---

## Cross-Project Gotchas

These are the things that will waste your time if you do not know them. Read this section carefully.

### 1. Agent YAML ownership confusion

Agent YAML configs live in `squidgy_updated_ui/agents/configs/` but are generated by `squidgy_agent_builder`. If you edit a YAML file manually in the frontend repo, the agent builder will not know about your changes. If the agent builder regenerates the file, your manual edits will be overwritten. Decide which system owns the file before editing.

### 2. Pia system prompt has three update points

When adding an agent, Pia's system prompt at `squidgy_updated_ui/knowledge_base/agents/personal_assistant/system_prompt.md` must be updated in three places:
- **Agent Relevance Rules** (~line 215): When Pia should mention the agent
- **Agent IDs** (~line 262): Display name to agent_id mapping
- **Routing Table** (~line 284): Keywords that trigger routing to the agent

The agent builder CLI handles this automatically. If you create an agent manually, you must update all three tables yourself.

### 3. build-agents.js is the sync gate

After creating or modifying agent configs, you MUST run `build-agents.js` in the frontend project. This script:
- Reads all YAML configs from `agents/configs/`
- Generates `agents.ts` (the typed agent registry used by the frontend)
- Syncs agent data to Supabase (`agents` and `personal_assistant_config` tables)

If you skip this step, the frontend and Supabase will not reflect your changes.

### 4. N8N credential limitation

The N8N REST API can create and update workflow definitions, but it CANNOT:
- Set credentials on nodes (must be done in N8N UI)
- Activate workflows (must be done in N8N UI)
- Set tags (must be done in N8N UI)

After deploying a workflow via the agent builder, you must manually configure credentials and activate it in the N8N web interface.

### 5. Two servers in the frontend project

The frontend project has an Express server (`server/`) that is NOT the Python backend. The architecture is:

```
Browser --> Express BFF (server/) --> Python Backend (Heroku)
                |
                +--> Supabase (direct)
                +--> N8N webhooks (direct)
```

The Express server handles proxying, static files, and lightweight routes. The Python backend handles AI processing, scraping, and file operations. Do not confuse them.

### 6. Chat bypasses the Python backend

Chat messages flow: Frontend --> N8N (directly). They do NOT pass through the Python backend. If you are debugging chat issues, look at:
- `client/lib/api.ts` (the `callN8NWebhook` function)
- The N8N workflow for the specific agent
- NOT the Python backend routes

### 7. Knowledge base queries DO go through the Python backend

Unlike chat messages, when an N8N agent needs to query the knowledge base during a conversation, it calls `POST /n8n/agent/query` on the Python backend. The backend performs semantic search in Neon and returns relevant chunks.

```
Chat message path:    Frontend ----> N8N (direct)
KB query path:        N8N ----> Python Backend ----> Neon
```

### 8. Post-deployment checklist for new N8N workflows

After deploying a new agent workflow to N8N:
1. Open N8N UI and find the new workflow
2. Set Supabase credentials on all Supabase nodes
3. Set OpenRouter credentials on the AI Agent node
4. Set any other required credentials
5. Activate the workflow
6. If you updated Pia's system prompt, restart Pia's N8N workflow so it picks up the new prompt

### 9. The "state" field naming trap

The conversation state field is called `state` everywhere. Not `conversation_state`, not `conversationState`, not `State`. Just `state`.

- Frontend sends: `{ body: { state: {...} } }`
- N8N Code node reads: `$input.first().json.body.state`
- N8N output parser schema uses: `state`
- Frontend receives: `{ state: {...} }`

But the status field IS capitalized: `Status` (capital S) with values `"Ready"` or `"Waiting"`.

### 10. Two databases for knowledge base

- **Supabase Storage**: Original uploaded files (PDF, DOCX, TXT blobs)
- **Neon pgvector**: Processed text chunks + vector embeddings

These are completely separate services with separate credentials. If file upload works but search does not, the issue is likely in the Neon pipeline, not Supabase.

---

## Debugging Checklist

When something breaks across projects, use this checklist to narrow down the problem.

### Agent not appearing in sidebar

1. Does the YAML config exist in `squidgy_updated_ui/agents/configs/{id}.yaml`?
2. Has `build-agents.js` been run after creating the YAML?
3. Does the agent exist in Supabase `agents` table?
4. Is the agent enabled in `personal_assistant_config` for this user?

### Chat not returning responses

1. Is the N8N workflow active? (Check N8N UI)
2. Are credentials set on all N8N nodes? (Check N8N UI)
3. Test with curl and check response body size:
   ```bash
   N8N_KEY=$(cat /Users/sethward/GIT/Squidgy/.n8n-api-key | tr -d '\n')
   curl -s -X POST https://n8n.theaiteam.uk/webhook/{agent_id} \
     -H "Content-Type: application/json" \
     -d '{"body":{"user_id":"test","user_mssg":"hello","session_id":"test","request_id":"test"}}' \
     | wc -c
   ```
   If the result is `0`, the Respond node never fired (workflow error).
4. Check the N8N execution log for the specific workflow.
5. Verify the webhook URL uses `/webhook/` (production) not `/webhook-test/`.

### Knowledge base search returning no results

1. Was the file uploaded to Supabase Storage? (Check Storage bucket)
2. Did the Python backend process it? (Check backend logs on Heroku)
3. Are there chunks in Neon? (Query the embeddings table directly)
4. Is the N8N workflow calling `/n8n/agent/query` on the correct backend URL?
5. Is the Python backend's `NEON_*` environment config correct?

### Website analysis failing

1. Is the Python backend running on Heroku?
2. Does the Express BFF proxy route work? (Test the Express endpoint directly)
3. Are Playwright dependencies installed on Heroku? (Check buildpacks)
4. Is the `PERPLEXITY_API_KEY` set in the backend environment?

---

## System Interaction Summary

```
                          +---------------------------+
                          |     squidgy_agent_builder |
                          |     (CLI Tool)            |
                          +--+-----+-----+-----+-----+
                             |     |     |     |
                   YAML      | N8N |     | Pia |  build-
                   config    | wf  |     | sys |  agents
                             | JSON|     | pmt |  .js
                             |     |     |     |
            +----------------v-----v-----v-----v-----------+
            |              squidgy_updated_ui               |
            |                                               |
            |  +----------+   +----------+   +-----------+  |
            |  | React    |   | Express  |   | Agent     |  |
            |  | SPA      |   | BFF      |   | Configs   |  |
            |  +----+-----+   +----+-----+   +-----------+  |
            +-------|--------------|------|------------------+
                    |              |      |
        +-----------+    +---------+     |
        |                |               |
   N8N webhooks    Proxy routes     Supabase
   (direct)        to backend       (direct)
        |                |               |
+-------v--------+ +----v-----------+ +-v-----------------+
|     N8N        | | squidgy_updated| |    Supabase        |
| Workflow       | | _backend       | | (Auth, Storage,    |
| Engine         | | (Python/Heroku)| |  Shared Tables)    |
|                | |                | +--------------------+
| - Agent chat   | | - Website      |
| - Scheduled    | |   analysis     | +--------------------+
|   backups      | | - KB processing| |    Neon DB          |
|                +--->KB queries    | | (pgvector           |
|                | | - GHL          | |  embeddings)        |
+----------------+ | - Embeddings ---->                    |
                   +----------------+ +--------------------+
```
