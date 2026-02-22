# Squidgy Agent Builder -- Brownfield Architecture Document

> **Audience:** Human developers onboarding to the project AND AI coding agents working on the codebase.
> **Last updated:** 2026-02-22
> **Status:** Living document -- update when architecture changes.

---

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-02-22 | Initial | Comprehensive brownfield analysis from source code audit |

---

## Quick Reference

### Commands

```bash
# Navigate to the project
cd /Users/sethward/GIT/Squidgy/squidgy_agent_builder

# Create a new agent (interactive CLI wizard)
npm run create

# Validate all YAML configs
npm run validate

# Validate a single config
npx tsx src/scripts/validate-agent.js agents/configs/brandy.yaml

# Build agents (run from squidgy_updated_ui)
cd ../squidgy_updated_ui && node scripts/build-agents.js

# Test an N8N webhook
N8N_KEY=$(cat /Users/sethward/GIT/Squidgy/.n8n-api-key | tr -d '\n')
curl -s -X POST https://n8n.theaiteam.uk/webhook/{agent_id} \
  -H "Content-Type: application/json" \
  -d '{"body":{"user_id":"test-123","user_mssg":"hello","session_id":"sess-1","request_id":"req-1"}}' \
  | python3 -m json.tool
```

### Key File Paths (Absolute)

| What | Path |
|------|------|
| Core service | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/src/services/agentBuilderService.ts` |
| CLI wizard | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/src/scripts/create-agent-interactive.js` |
| Validator | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/src/scripts/validate-agent.js` |
| YAML template | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/templates/agent_config.yaml` |
| KB template | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/templates/instructions.md` |
| DB template | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/templates/database_schema.sql` |
| DB insert template | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/templates/INSERT_AGENT_TO_DATABASE.sql` |
| Brandy reference | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/examples/brandy/` |
| N8N API key | `/Users/sethward/GIT/Squidgy/.n8n-api-key` |
| package.json | `/Users/sethward/GIT/Squidgy/squidgy_agent_builder/package.json` |

### Output Destinations (Where Generated Files Land)

| Artifact | Destination |
|----------|-------------|
| Agent YAML config | `squidgy_updated_ui/agents/configs/{agent_id}.yaml` |
| N8N workflow JSON | `squidgy_updated_ui/n8n/{agent_id}_workflow.json` |
| Pia system prompt | `squidgy_updated_ui/knowledge_base/agents/personal_assistant/system_prompt.md` |
| Knowledge base files | `squidgy_updated_ui/knowledge_base/agents/{agent_id}/` |

---

## 1. Project Overview

The Squidgy Agent Builder is a TypeScript CLI tool that generates complete AI agent pipelines for the Squidgy platform. Given interactive user input, it produces:

1. A YAML configuration file defining the agent's identity, personality, and UI settings
2. An N8N workflow JSON implementing the proven 10-node AI pattern
3. Updates to Pia's (the personal assistant) system prompt for routing
4. API deployment of the workflow to the N8N instance

The reference implementation is **Brandy** (Brand Advisor), located at `examples/brandy/`.

### What This Project Is NOT

- It is NOT a web application or API server. It is a CLI tool.
- It does NOT run the agents. N8N runs the agents.
- It does NOT manage the Supabase database schema. That is manual.
- It does NOT set N8N credentials. That is manual (UI only).

---

## 2. Tech Stack

| Category | Technology | Version | Notes |
|----------|-----------|---------|-------|
| Runtime | Node.js | ES Module | `"type": "module"` in package.json |
| Language | TypeScript | via tsx 4.7.0 | Runtime execution, not compiled |
| YAML | js-yaml | ^4.1.0 | Config parsing and generation |
| CLI | readline (Node built-in) | -- | Interactive prompts (not inquirer at runtime -- see Known Issues) |
| Terminal | ANSI escape codes | -- | Colored output (chalk is a dependency but CLI uses raw ANSI) |
| Spinners | ora | ^8.0.1 | Listed as dependency, not currently used in CLI |

### Dependencies vs Actual Usage (Important for AI Coders)

The `package.json` lists `inquirer`, `chalk`, and `ora` as dependencies, but the CLI wizard (`create-agent-interactive.js`) actually uses Node's built-in `readline` and raw ANSI escape codes. The listed npm dependencies are available but not imported in the main scripts. This matters if you are adding features -- you can use them, but the existing code does not.

---

## 3. Project Structure

```
squidgy_agent_builder/
|
|-- package.json                          # ES Module config, npm scripts
|
|-- src/
|   |-- services/
|   |   `-- agentBuilderService.ts        # Core service (1198 lines) -- ALL generation logic
|   |
|   `-- scripts/
|       |-- create-agent-interactive.js   # CLI wizard (536 lines) -- user-facing entry point
|       `-- validate-agent.js             # YAML validator (270 lines) -- pre-deploy check
|
|-- templates/
|   |-- agent_config.yaml                 # YAML skeleton for new agents
|   |-- instructions.md                   # Knowledge base template
|   |-- database_schema.sql               # Supabase table template
|   `-- INSERT_AGENT_TO_DATABASE.sql      # SQL insert template
|
|-- examples/
|   `-- brandy/                           # THE reference implementation
|       |-- config/brandy.yaml            # Brandy's YAML config
|       |-- knowledge_base/               # instructions.md, wizard_flow.md, brand_methodology.md
|       |-- database/brands_table.sql     # Brandy's Supabase schema
|       `-- n8n/brandy_workflow.json      # Brandy's N8N workflow
|
`-- docs/                                 # 15+ guides (accumulated during development)
    |-- AGENT_SETUP_COMPLETE_GUIDE.md     # 27KB deep-dive
    |-- BRANDY_REFERENCE_IMPLEMENTATION.md
    |-- BRANDY_DEPLOYMENT_COMPLETE.md
    |-- DEPLOY_BRANDY.md
    |-- AGENT_BUILDER_QUICKSTART.md
    |-- AGENT_BUILDER_README.md
    |-- AGENT_BUILDER_STATUS.md
    |-- AGENT_BUILDER_SUMMARY.md
    |-- ...
    `-- n8n/                              # N8N-specific documentation
        |-- N8N_AGENT_PATTERNS_DIAGRAM.md
        |-- N8N_AGENT_TROUBLESHOOTING.md
        |-- N8N_AGENT_BUILDER_GUIDE.md
        |-- N8N_AGENT_QUICK_REFERENCE.md
        |-- N8N_AGENT_DOCUMENTATION_INDEX.md
        `-- N8N_GITHUB_BACKUP_WORKFLOW.md
```

### File Ownership Rules

The agent builder generates files into `squidgy_updated_ui/` (a sibling project). It navigates up one directory (`..`) to find it. The `saveAgent()` method in `agentBuilderService.ts` writes to paths relative to `process.cwd()`, which is expected to be the `squidgy_agent_builder/` directory. If you run the CLI from a different working directory, file placement will break.

---

## 4. Core Architecture

### 4.1 AgentBuilderService (Singleton)

**File:** `src/services/agentBuilderService.ts` (1198 lines)
**Pattern:** Singleton via `getInstance()`

This is the brain of the entire system. Every generation method lives here. There is no dependency injection, no plugin system, no external configuration beyond the conversation object passed in.

#### Public API

| Method | Purpose | Returns |
|--------|---------|---------|
| `detectTier(conversation)` | Determines agent complexity (1-4) | `1 \| 2 \| 3 \| 4` |
| `generateAgentId(name)` | Converts display name to snake_case ID | `string` |
| `generateYAML(conversation)` | Produces complete YAML config string | `string` |
| `generateN8NWorkflow(conversation)` | Produces N8N workflow JSON object | `object` |
| `updatePiaSystemPrompt(conversation)` | Modifies Pia's system_prompt.md in-place | `boolean` |
| `deployToN8N(workflow, apiKey, baseUrl)` | POSTs workflow to N8N API | `Promise<DeployResult>` |
| `saveAgent(conversation)` | Full pipeline orchestrator (calls all above) | `Promise<GeneratedAgentConfig>` |

#### Key Private Methods

| Method | What It Generates |
|--------|-------------------|
| `generateAIPoweredWorkflow()` | The 10-node N8N workflow JSON |
| `generateAIPrepareDataCode()` | JavaScript for the Code node (data merging) |
| `generateAIAgentSystemPrompt()` | System prompt for the AI Agent node (1000+ words) |
| `generateOutputParserSchema()` | JSON schema for the Structured Output Parser |
| `generateRespondBody()` | Response template for Respond to Webhook nodes |
| `insertTableRow()` | Markdown table row insertion for Pia updates |

#### Tier Detection Logic

```
Tier 4: needsCustomUI === true
Tier 3: integrations include calculator/maps/external_api/regional_config
Tier 2: platforms.length > 0 OR integrations.length > 1
Tier 1: Everything else (basic chat)
```

Each tier adds features:
- Tier 1: `text_input`, `suggestion_buttons`
- Tier 2: adds `file_upload`
- Tier 3: adds `voice_input`
- Tier 4: adds Figma-based custom UI config

### 4.2 CLI Wizard

**File:** `src/scripts/create-agent-interactive.js` (536 lines)
**Runtime:** Plain JavaScript (not TypeScript), executed via `tsx`

The wizard is a sequential state machine with these phases:

```
welcome -> purpose -> category -> naming -> personality -> capabilities ->
integrations -> ui -> ai_config -> pia_config -> summary -> generate
```

Each phase is an `async function` that collects input, stores it on a `conversation` object, then calls the next phase. There is no back-navigation -- if the user wants to change something, they must restart.

The `generateAgent()` function at the end calls `builder.saveAgent(conversation)` which runs the full pipeline.

### 4.3 Validator

**File:** `src/scripts/validate-agent.js` (270 lines)

Validates YAML configs against these rules:

| Rule | Severity |
|------|----------|
| `agent.id`, `name`, `category`, `description`, `webhook_url` must exist | ERROR |
| `agent.id` must be `[a-z0-9_]+` only | ERROR |
| Category must be one of 6 valid values | ERROR |
| `/webhook-test/` in URL (dev-only, will break in production) | ERROR |
| Filename must match `{agent.id}.yaml` | WARNING |
| Missing capabilities, suggestions, or initial_message | WARNING |
| Unknown interface features | WARNING |

Can validate a single file or all files in `agents/configs/`.

---

## 5. The N8N 10-Node Workflow Pattern

This is the proven pattern that every generated agent uses. It was battle-tested with Brandy and is now hardcoded into the generator.

### Flow Diagram

```
[Webhook] --> [Supabase getAll] --> [Code: Prepare Data] --> [AI Agent]
                                                                |
                                                    +-----------+-----------+
                                                    |           |           |
                                              [OpenRouter] [Memory]  [Output Parser]
                                                                |
                                                          [If Ready]
                                                          /        \
                                                    true /          \ false
                                                        /            \
                                              [Respond Ready]  [Respond Waiting]
```

### Node Details

| # | Node | Type | Key Config |
|---|------|------|------------|
| 1 | Webhook | `n8n-nodes-base.webhook` | POST, `responseMode: 'responseNode'` |
| 2 | Supabase | `n8n-nodes-base.supabase` | `operation: 'getAll'`, `alwaysOutputData: true`, `onError: 'continueRegularOutput'` |
| 3 | Code - Prepare Data | `n8n-nodes-base.code` | Merges webhook body + Supabase data + state |
| 4 | AI Agent | `@n8n/n8n-nodes-langchain.agent` | System prompt with mustache variables |
| 5 | OpenRouter Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | `model: 'anthropic/claude-3-5-sonnet'` |
| 6 | Simple Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | `contextWindowLength: 100`, keyed by session_id |
| 7 | Structured Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | `autoFix: true` |
| 8 | If Ready | `n8n-nodes-base.if` | Checks `$json.output.Status === 'Ready'` |
| 9 | Respond - Waiting | `n8n-nodes-base.respondToWebhook` | `respondWith: 'json'`, `typeVersion: 1.1` |
| 10 | Respond - Ready | `n8n-nodes-base.respondToWebhook` | Same as above, different status |

### Output Parser Schema (Strict Format)

```json
{
  "response": "The conversational message to the user (markdown allowed)",
  "Status": "Waiting",
  "state": {
    "phase": "assessment",
    "data_exists": false,
    "wizard_step": 0,
    "wizard_data": {}
  }
}
```

**CRITICAL:** `Status` uses a capital "S". The frontend and If Ready node both depend on this exact casing.

### Respond Body Template

Both Respond nodes use this pattern:
```
Code node for IDs:     $('Code - Prepare Data').item.json.user_id
AI Agent for output:   $('AI Agent - {Name}').item.json.output.response
```

This split is intentional -- the Code node holds the stable identifiers, while the AI Agent holds the generated content.

---

## 6. Data Flow

### 6.1 Agent Creation Pipeline

```
User Input (CLI)
    |
    v
AgentBuilderConversation object
    |
    +---> generateYAML()     ---> agents/configs/{id}.yaml
    |
    +---> generateN8NWorkflow() ---> n8n/{id}_workflow.json
    |                                    |
    |                                    +---> deployToN8N() ---> N8N API (POST /api/v1/workflows)
    |
    +---> updatePiaSystemPrompt() ---> Modifies system_prompt.md (3 tables)
    |
    +---> generateSetupGuide() ---> Console output
    |
    v
build-agents.js (manual step)
    |
    +---> Reads all YAML configs
    +---> Generates agents.ts
    +---> Syncs to Supabase (personal_assistant_config + agents tables)
```

### 6.2 Runtime Data Flow (When Agent Is Live)

```
Frontend chat message
    |
    v
POST /webhook/{agent_id}
Body: { user_id, user_mssg, session_id, request_id, state }
    |
    v
[Supabase getAll] -- fetches existing user data (may return empty [])
    |
    v
[Code - Prepare Data] -- merges webhook + DB + state into flat JSON
    |
    v
[AI Agent] -- processes with system prompt, memory, output parser
    |
    v
[If Ready] -- branches on Status field
    |
    v
[Respond] -- returns JSON to frontend
Response: { user_id, session_id, agent_name, agent_response, agent_status, state }
```

### 6.3 The Conversation State Lifecycle

```
Frontend sends:   { body: { state: { phase: "wizard", wizard_step: 2, wizard_data: {...} } } }
                     |
Code node reads:     $input.first().json.body.state    (NOT conversation_state)
                     |
AI Agent receives:   {{ $json.conversation_state }}     (stringified JSON in prompt)
                     |
AI Agent outputs:    { state: { phase: "wizard", wizard_step: 3, wizard_data: {...} } }
                     |
Respond node sends:  {{ JSON.stringify($('AI Agent').item.json.output.state) }}
                     |
Frontend stores and resends on next message
```

---

## 7. Interfaces and Type Definitions

### AgentBuilderConversation (Core Input Type)

```typescript
interface AgentBuilderConversation {
  // Basic Info
  purpose?: string;
  category?: 'MARKETING' | 'SALES' | 'HR' | 'SUPPORT' | 'OPERATIONS' | 'GENERAL';
  name?: string;
  id?: string;

  // Personality
  tone?: string;
  style?: string;
  approach?: string;

  // Capabilities
  capabilities?: string[];
  platforms?: string[];
  integrations?: string[];

  // UI
  needsCustomUI?: boolean;
  figmaUrl?: string;

  // Complexity
  detectedTier?: 1 | 2 | 3 | 4;

  // Wizard Support
  hasConditionalWizard?: boolean;
  wizardPhases?: WizardPhase[];
  conversationStateSchema?: ConversationStateSchema;
  databaseSchema?: DatabaseSchema;
  knowledgeBaseFiles?: KnowledgeBaseFile[];
  importCapabilities?: ImportCapability[];

  // AI Configuration
  usesAI?: boolean;
  usesConversationState?: boolean;
  supabaseTable?: string;
  systemPromptContent?: string;

  // Pia Routing
  piaRelevanceRule?: string;
  piaRoutingKeywords?: string;
  piaDisplayName?: string;

  // State Machine
  phase?: string;
}
```

### DeployResult (N8N API Response)

```typescript
interface DeployResult {
  workflowId?: string;
  webhookUrl?: string;
  success: boolean;
  error?: string;
  manualSteps: string[];
}
```

### GeneratedAgentConfig (Pipeline Output)

```typescript
interface GeneratedAgentConfig {
  yamlPath: string;
  yamlContent: string;
  n8nWorkflowPath?: string;
  n8nWorkflowContent?: string;
  integrationScripts?: string[];
  setupGuide: string;
  tier: number;
}
```

---

## 8. N8N API Integration

### What the API CAN Do

- Create new workflows (`POST /api/v1/workflows`)
- Accepts: `name`, `nodes`, `connections`, `settings`
- Returns the created workflow with its assigned `id`

### What the API CANNOT Do (UI Only)

| Action | Must Do In N8N UI |
|--------|-------------------|
| Set credentials | Select OpenRouter + Supabase credentials per node |
| Activate workflow | Click "Publish" button |
| Set tags | Add tags via UI |
| Move to folder | Drag to Squidgy folder |
| Set active status | Toggle active/inactive |

### Authentication

```
Header: X-N8N-API-KEY: {key}
Key location: /Users/sethward/GIT/Squidgy/.n8n-api-key
Base URL: https://n8n.theaiteam.uk
```

---

## 9. Pia System Prompt Updates

The agent builder modifies Pia's system prompt in-place at:
`squidgy_updated_ui/knowledge_base/agents/personal_assistant/system_prompt.md`

It updates three markdown tables by finding section headers and appending rows:

| Table | Section Header to Search | What Gets Added |
|-------|--------------------------|-----------------|
| Agent Relevance Rules | `"Agent Relevance Rules"` | `\| Display Name \| Show When \|` |
| Agent IDs | `"AGENT IDs"` | `\| Display Name \| \`agent_id\` \|` |
| Routing | `"ROUTING"` | `\| Keywords \| Route To \|` |

### How `insertTableRow()` Works

1. Searches for the section header string in all lines
2. Finds the last `|`-prefixed row in the table after that header
3. Splices the new row after the last existing row
4. Checks for duplicates by searching for the `agentId` string first

### After Updating Pia

You MUST restart the N8N Personal Assistant workflow. Pia caches her system prompt, so changes are invisible until the workflow reloads.

---

## 10. Validation Rules

The validator (`validate-agent.js`) enforces these rules:

### Errors (Block Deployment)

| Rule | Regex/Check |
|------|-------------|
| Required fields exist | `agent.id`, `agent.name`, `agent.category`, `agent.description`, `n8n.webhook_url` |
| ID format | `/^[a-z0-9_]+$/` -- lowercase, numbers, underscores only |
| Valid category | One of: `MARKETING`, `SALES`, `HR`, `SUPPORT`, `OPERATIONS`, `GENERAL` |
| No test webhooks | Rejects URLs containing `/webhook-test/` |

### Warnings (Non-Blocking)

| Rule | Check |
|------|-------|
| Filename matches ID | `{agent.id}.yaml` must equal the filename |
| Capabilities defined | `agent.capabilities` should have entries |
| Suggestions defined | `suggestions` array should have entries |
| Initial message exists | `agent.initial_message` should be set |
| Known interface features | Only `text_input`, `file_upload`, `voice_input`, `suggestion_buttons`, `calculator_widget`, `map_integration` |
| Known personality values | Warns on unusual tone/style/approach values |

---

## 11. Tribal Knowledge and Gotchas

This section captures hard-won lessons from production debugging. Read every item before making changes.

### N8N Workflow Gotchas

1. **NEVER use `/webhook-test/` in YAML configs.** Test webhooks only exist while the N8N editor is open. Production must use `/webhook/`. The validator catches this.

2. **`Status` must have a capital "S" in the output parser.** The If Ready node checks `$json.output.Status`. If you use lowercase `status`, the If node will never match and every response will go to the "Waiting" branch.

3. **Supabase `get` vs `getAll`.** The `get` operation throws an error when no row exists (crashing the entire workflow). Always use `getAll` with `limit: 1` and `condition: 'eq'`. Combined with `alwaysOutputData: true`, this guarantees downstream nodes always execute.

4. **`alwaysOutputData: true` is non-negotiable on Supabase nodes.** Without it, an empty result produces zero output items, and every downstream node silently stops. The workflow appears to hang -- no error, no response, nothing.

5. **`onError: 'continueRegularOutput'` on Supabase nodes.** Without this, a Supabase error kills the entire workflow. With it, errors are swallowed and an empty result is passed downstream (which the Code node handles gracefully).

6. **Output responses must stay under 1500 words.** The Structured Output Parser can overflow on very long responses. The system prompt includes an explicit instruction to split long content across multiple turns.

7. **Respond node `typeVersion` must be `1.1`.** Version 1.0 has different response body handling. Always use `1.1` with `respondWith: 'json'` and explicit `responseBody`.

### Frontend Integration Gotchas

8. **Frontend sends `state`, not `conversation_state`.** The Code node reads `$input.first().json.body.state`. If you look for `conversation_state` you will get `undefined`. This field name mismatch is a historical artifact.

9. **`uses_conversation_state: true` in YAML is required.** Without this flag in the agent config, the frontend will not send the `state` field in the webhook body.

### Deployment Gotchas

10. **N8N API creates workflows as INACTIVE.** After `deployToN8N()` succeeds, the workflow exists but does not respond to webhooks. You must open the N8N UI and click "Publish".

11. **After updating Pia's system prompt, restart the N8N Personal Assistant workflow.** Pia caches the prompt. Changes are invisible until cache invalidation.

12. **Dev vs Live database confusion.** Brandy deployment issues were caused by the Code node querying the wrong Supabase database. Always verify the Supabase credential in N8N points to the correct environment.

13. **`build-agents.js` syncs to TWO tables.** It updates both `personal_assistant_config` AND `agents` in Supabase. If one sync fails, the agent may be partially visible.

### Code and Architecture Gotchas

14. **Import path mismatch in CLI wizard.** The file `create-agent-interactive.js` imports from `'../server/services/agentBuilderService.ts'` but the actual file is at `src/services/agentBuilderService.ts` (no `server` directory in the path). This may work via tsx resolution or a symlink. If you move files, this import will break.

15. **`saveAgent()` uses `process.cwd()` for file paths.** The CLI must be run from the `squidgy_agent_builder/` directory. Running from the project root or any other directory will write files to wrong locations.

16. **The `inquirer`, `chalk`, and `ora` dependencies are installed but not used.** The CLI wizard uses Node's built-in `readline` and raw ANSI codes instead. These packages are available if you want to use them for new features.

17. **Brandy is THE reference implementation.** When in doubt about how something should work, check `examples/brandy/`. It is the only fully deployed and tested agent built with this tool.

18. **Templates exist but are not used by the generator.** The `templates/` directory contains `agent_config.yaml`, `instructions.md`, `database_schema.sql`, and `INSERT_AGENT_TO_DATABASE.sql`. These are reference templates for manual use. The `agentBuilderService.ts` generates everything from scratch via code, not from these templates.

---

## 12. Cross-Project Dependencies

The agent builder does not exist in isolation. It writes to and reads from sibling projects.

```
squidgy_agent_builder/
    |
    |-- WRITES TO --> squidgy_updated_ui/agents/configs/{id}.yaml
    |-- WRITES TO --> squidgy_updated_ui/n8n/{id}_workflow.json
    |-- MODIFIES  --> squidgy_updated_ui/knowledge_base/agents/personal_assistant/system_prompt.md
    |-- CALLS     --> N8N API at https://n8n.theaiteam.uk
    |
    |-- TRIGGERS  --> squidgy_updated_ui/scripts/build-agents.js
    |                    |
    |                    +-- READS  --> squidgy_updated_ui/agents/configs/*.yaml
    |                    +-- WRITES --> squidgy_updated_ui/shared/agents.ts (or similar)
    |                    +-- SYNCS  --> Supabase (personal_assistant_config + agents tables)
```

### Assumptions

- The sibling directory `squidgy_updated_ui/` exists at `../squidgy_updated_ui/` relative to the agent builder
- The Pia system prompt file exists at the expected path
- The N8N instance is accessible at `https://n8n.theaiteam.uk`
- The N8N API key file exists at `/Users/sethward/GIT/Squidgy/.n8n-api-key`

---

## 13. Reference Implementation: Brandy

Brandy (Brand Advisor) is the only fully deployed agent built with this system. Use it as ground truth.

**Location:** `examples/brandy/`

| File | Purpose |
|------|---------|
| `config/brandy.yaml` | Complete YAML config with wizard phases, database schema, knowledge base references |
| `knowledge_base/instructions.md` | Agent instructions and personality |
| `knowledge_base/wizard_flow.md` | 6-element punk branding wizard steps |
| `knowledge_base/brand_methodology.md` | Domain knowledge for brand strategy |
| `database/brands_table.sql` | Supabase table schema for brand data |
| `n8n/brandy_workflow.json` | The actual deployed N8N workflow |

### Brandy's Active N8N Workflow

- **Workflow ID:** `4mUkvAI8vgjLttpT` (v2, working)
- **Previous versions:** `D7hqn6HStORy687Y` (v1, broken), `EuCOoyT1Tk22hm68` (JS-only, inactive)
- **Branding model:** 6-element punk branding (Atmosphere, Rebellious Edge, Enemy, Visual Direction, Hook Style, Voice)
- **Behavior:** Hybrid -- wizard for new users, advisor for returning users

---

## 14. Extending the System

### Adding a New Agent (The Happy Path)

1. Run `npm run create` from the `squidgy_agent_builder/` directory
2. Follow the 9-step wizard
3. Open the created N8N workflow in the UI
4. Set OpenRouter and Supabase credentials
5. Click Publish
6. Run `build-agents.js` from `squidgy_updated_ui/`
7. Test with curl
8. Restart Pia's N8N workflow if routing was updated

### Adding a New Wizard Phase to an Existing Agent

Modify the agent's YAML config to add `wizardPhases` entries, then regenerate the N8N workflow. The system prompt generator will automatically include wizard steps.

### Adding a New Node Type to the Workflow Pattern

Edit `generateAIPoweredWorkflow()` in `agentBuilderService.ts`. Add the node to the `nodes` array and update the `connections` object. Be extremely careful with N8N connection types -- LangChain sub-nodes use `ai_languageModel`, `ai_memory`, and `ai_outputParser` connection types, not `main`.

### Modifying the Output Parser Schema

Edit `generateOutputParserSchema()` in `agentBuilderService.ts`. Any changes here must be mirrored in:
- The If Ready node condition (`$json.output.Status`)
- The Respond node body templates
- The frontend's response parsing logic

---

## 15. Known Limitations

1. **No back-navigation in the CLI wizard.** Users must restart to change earlier answers.
2. **No workflow versioning.** Each deployment creates a new workflow. Old versions are not cleaned up.
3. **No automated testing.** There are no unit tests, integration tests, or CI/CD.
4. **No dry-run mode.** The pipeline always writes files and potentially deploys to N8N.
5. **Single-agent generation only.** Cannot batch-create multiple agents.
6. **Hardcoded N8N base URL.** The URL `https://n8n.theaiteam.uk` appears in multiple places.
7. **No rollback mechanism.** If deployment partially fails (e.g., YAML written but N8N deploy fails), there is no way to undo the changes automatically.
8. **Templates are decorative.** The `templates/` directory files are not used by the code generator.

---

## 16. Documentation Index

The `docs/` directory contains extensive guides accumulated during development. Key documents:

| Document | Use When |
|----------|----------|
| `AGENT_SETUP_COMPLETE_GUIDE.md` | Deep-dive into the full setup process (27KB) |
| `BRANDY_REFERENCE_IMPLEMENTATION.md` | Understanding how Brandy was built |
| `BRANDY_DEPLOYMENT_COMPLETE.md` | Post-deployment verification steps |
| `AGENT_BUILDER_QUICKSTART.md` | Getting started fast |
| `docs/n8n/N8N_AGENT_PATTERNS_DIAGRAM.md` | Visual guide to the 10-node pattern |
| `docs/n8n/N8N_AGENT_TROUBLESHOOTING.md` | Debugging N8N workflow issues |
| `docs/n8n/N8N_AGENT_BUILDER_GUIDE.md` | N8N-specific builder documentation |

---

## 17. For AI Coding Agents

If you are an AI agent working on this codebase, here are the rules:

1. **Read `examples/brandy/` before making changes.** It is the only proven implementation.
2. **Never modify `agentBuilderService.ts` without understanding the full pipeline.** A change to one generator method can break the entire chain.
3. **Test N8N changes with curl, not the UI.** Check the response body size -- 0 bytes means the Respond node never fired.
4. **Do not create files in the project root.** See the file placement rules in the project's `CLAUDE.md`.
5. **The `Status` field is capital-S `Status`.** This is not a typo. Do not "fix" it.
6. **The `state` field (not `conversation_state`) is what the frontend sends.** This is not a typo. Do not "fix" it.
7. **If you add a new Supabase node, always use `getAll` + `alwaysOutputData: true` + `onError: 'continueRegularOutput'`.** No exceptions.
8. **Run `npm run validate` after modifying any YAML config.** It catches common errors.
9. **After any Pia system prompt change, note that the N8N Personal Assistant workflow must be restarted.** Add this to your output/instructions.
10. **The N8N API key is at `/Users/sethward/GIT/Squidgy/.n8n-api-key`.** Do not hardcode it. Do not commit it.
