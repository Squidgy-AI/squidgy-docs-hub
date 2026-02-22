# Squidgy Frontend (squidgy_updated_ui) -- Brownfield Architecture Document

| Field | Value |
|-------|-------|
| **Document Type** | Brownfield Architecture |
| **Project** | squidgy_updated_ui |
| **Audience** | Human developers onboarding, AI coding agents |
| **Last Updated** | 2026-02-22 |

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-02-22 | Claude Opus 4.6 | Initial comprehensive brownfield architecture document |

---

## 1. System Overview

Squidgy is a full-stack React/Express single-page application for managing multi-agent AI assistants. Users onboard through a guided wizard, configure AI agents tailored to their business, and interact with those agents through a unified chat interface. The system orchestrates agent conversations through N8N workflows, persists data in Supabase, and integrates with GoHighLevel for CRM functionality.

The frontend is a monorepo containing both the React SPA client and an Express.js API server. They share types via a `shared/` directory and are built together through two separate Vite configurations. In development, the Express server runs as Vite middleware; in production, it runs as a standalone Node.js process.

**What this application does, in plain terms:** A business owner signs up, pastes their website URL, and the system analyzes their company. It then recommends AI agents (brand advisor, newsletter writer, social media scheduler, etc.) that the user can enable and chat with. Each agent runs as an N8N workflow that receives messages via webhook and returns structured responses.

---

## 2. Quick Reference

### 2.1 Essential Commands

```bash
# Development
pnpm install                    # Install dependencies (MUST use pnpm, not npm)
npm run dev                     # Vite dev server + Express on port 8080
npm run dev:watch               # Vite only, no agent build step

# Production
npm run build                   # Full build: client (dist/spa/) + server (dist/server/)
npm run start                   # Run production server from dist/server/node-build.mjs

# Type checking
npm run typecheck               # tsc (note: strict: false, so this is lenient)

# Tests
npm run test                    # Vitest (minimal coverage currently)

# Agent management
npm run agent:validate          # Validate YAML agent configs
node scripts/build-agents.js   # YAML -> agents.ts + Supabase sync (runs on every build)
```

### 2.2 Key File Paths

| Purpose | Path |
|---------|------|
| App entry point | `client/App.tsx` |
| Vite client config | `vite.config.ts` |
| Vite server config | `vite.config.server.ts` |
| Express server entry | `server/index.ts` |
| Supabase client | `client/lib/supabase.ts` |
| TypeScript config | `tsconfig.json` |
| Agent YAML configs | `agents/configs/*.yaml` |
| Agent knowledge bases | `knowledge_base/agents/{agent_id}/` |
| N8N workflow backups | `n8n/` |
| Database schemas | `database/schemas/` |
| Build scripts | `scripts/` |
| Shared types | `shared/api.ts` |
| Deployment config | `vercel.json` |
| Package manifest | `package.json` |

### 2.3 Environment Variables

The application requires these `VITE_` prefixed variables (client-visible) and server-side variables:

| Variable | Used By | Purpose |
|----------|---------|---------|
| `VITE_SUPABASE_URL` | Client + Server | Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Client + Server | Supabase anonymous key |
| `PING_MESSAGE` | Server | Health check message |
| `N8N_LOAD_INSTRUCTIONS_URL` | Scripts | Webhook for syncing instructions |

If `VITE_SUPABASE_URL` or `VITE_SUPABASE_ANON_KEY` are missing, the Supabase client silently falls back to dummy values (`https://dummy.supabase.co` / `dummy-key`). This prevents crashes during local development but means all Supabase calls will fail silently.

---

## 3. Tech Stack

| Category | Technology | Version | Notes |
|----------|-----------|---------|-------|
| Runtime | Node.js | v22 target | ES2020 compile target in tsconfig |
| Package Manager | pnpm | 10.14.0 | Enforced via `packageManager` field in package.json |
| Framework | React | 18.3.1 | SPA with React Router 6 |
| Build | Vite | 7.1.2 | SWC plugin (`@vitejs/plugin-react-swc`) for fast HMR |
| Language | TypeScript | 5.9.2 | `strict: false`, `noImplicitAny: false` |
| Styling | TailwindCSS | 3.4.17 | + typography plugin, animate plugin |
| UI Components | Radix UI | 27 primitives | Wrapped as shadcn/ui-style components in `client/components/ui/` |
| Icons | Lucide React | 0.539.0 | |
| State / Data Fetching | TanStack React Query | 5.84.2 | |
| Forms | React Hook Form | 7.62.0 | + Zod 3.25.76 for validation |
| HTTP Client | Axios | 1.12.2 | |
| Animation | Framer Motion | 12.23.12 | |
| Charts | Recharts | 2.12.7 | |
| 3D | Three.js / React Three Fiber | 0.176.0 / 8.18.0 | Used sparingly |
| DB Client | Supabase JS | 2.57.0 | PKCE auth flow |
| Server | Express | 5.1.0 | Express 5 (not 4) |
| PDF Generation | jsPDF | 3.0.3 | |
| HTML Parsing | Cheerio | 1.1.2 | Server-side website analysis |
| Browser Automation | Playwright | 1.56.0 | GHL integration helper |
| Analytics | PostHog | 1.277.0 | |
| Testing | Vitest | 3.2.4 | Minimal coverage |

---

## 4. Architecture

### 4.1 High-Level Data Flow

```
User Browser (React SPA)
    |
    |-- Direct Supabase calls (auth, CRUD via supabase-js)
    |-- API calls to Express server (/api/*)
    |-- N8N webhook calls (via client services)
    |
    v
Express Server (port 8080 / Vercel serverless)
    |-- /website/* -> Cheerio + Playwright for site analysis
    |-- /ghl/* -> GoHighLevel CRM API
    |-- /agents/* -> Agent management
    |-- /storage/* -> Supabase storage proxy (masks URLs)
    |-- /api/google/calendar/* -> Google Calendar OAuth
    |
Supabase (PostgreSQL + Auth + Storage)
    |-- Auth: PKCE flow, auto-refresh, session persistence
    |-- Tables: profiles, agents, brands, website_analysis, ...
    |-- Storage: File uploads proxied through Express
    |
N8N (External, self-hosted at n8n.theaiteam.uk)
    |-- Each agent has a webhook endpoint
    |-- 10-node pattern: Webhook -> Supabase -> Code -> AI Agent -> If -> Respond
    |-- Returns structured JSON: { response, Status, state }
```

### 4.2 Project Structure

```
squidgy_updated_ui/
|
+-- agents/configs/              # YAML agent definitions (10 files, 7 distinct agents)
|
+-- client/                      # React SPA (the "frontend")
|   +-- App.tsx                  # Root component, all routes defined here
|   +-- global.css               # Global styles
|   +-- pages/                   # 47 route pages
|   |   +-- admin/               # Admin dashboard pages (4)
|   |   +-- agents/              # Dynamic agent pages
|   |   +-- mobile/              # Mobile-optimized pages
|   |   +-- new_onboarding/      # Latest onboarding flow
|   |   +-- onboarding/          # Original onboarding flow (7 pages)
|   |   +-- referrals/           # Referral system
|   |   +-- *.tsx                # Top-level pages (Dashboard, Login, ChatPage, etc.)
|   |
|   +-- components/              # React components
|   |   +-- chat/                # 21 chat-related components
|   |   +-- ui/                  # 52 Radix wrapper components (shadcn/ui pattern)
|   |   +-- layout/              # Sidebar, Header
|   |   +-- mobile/              # Mobile touch interfaces
|   |   +-- onboarding/          # ComprehensiveOnboarding wizard
|   |   +-- billing/             # Billing components
|   |   +-- media/               # Media handling
|   |   +-- modals/              # Modal dialogs
|   |   +-- referrals/           # Referral components
|   |   +-- *.tsx                # Top-level components (ChatInterface, DynamicAgentPage, etc.)
|   |
|   +-- services/                # 29 business logic services
|   +-- hooks/                   # 10 custom React hooks
|   +-- lib/                     # Core libraries (supabase.ts, api.ts)
|   +-- types/                   # TypeScript interfaces
|   +-- contexts/                # React Context providers (SidebarContext)
|   +-- routes/                  # Route configuration
|   +-- styles/                  # CSS modules and mobile styles
|
+-- server/                      # Express.js API backend
|   +-- index.ts                 # Express app factory (createServer)
|   +-- node-build.ts            # Production server entry
|   +-- routes/                  # 6 route modules
|   |   +-- agents.ts            # Agent CRUD
|   |   +-- website.ts           # Website analysis (Cheerio + screenshots)
|   |   +-- ghl.ts               # GoHighLevel CRM integration
|   |   +-- storage-proxy.ts     # Supabase storage URL masking
|   |   +-- googleCalendar.ts    # Google Calendar OAuth
|   |   +-- demo.ts              # Demo endpoint
|   +-- services/                # Server-side services
|
+-- shared/                      # Shared between client and server
|   +-- api.ts                   # Shared type definitions
|
+-- knowledge_base/agents/       # Agent system prompts and instructions
|   +-- personal_assistant/      # Pia: system_prompt.md, instructions.md, onboarding_flow.md
|   +-- brandy/                  # Brandy: instructions.md, wizard_flow.md, brand_methodology.md
|   +-- [other agents]/
|
+-- n8n/                         # N8N workflow JSON backups (deployed versions)
+-- database/                    # SQL schemas and migrations
|   +-- schemas/                 # Table definitions
|   +-- migrations/              # Migration scripts
+-- scripts/                     # Build and agent management scripts
|   +-- build-agents.js          # YAML -> TypeScript + Supabase sync
|   +-- validate-agent.js        # YAML config validation
+-- public/                      # Static assets (avatars, images)
```

### 4.3 Module Dependency Graph (Simplified)

```
App.tsx
  +-- QueryClientProvider (TanStack React Query)
  +-- UserProvider (auth state via useUser hook)
  +-- MobileProvider (responsive detection)
  +-- TooltipProvider (Radix)
  +-- BrowserRouter (React Router 6)
      +-- AuthHandler (redirects for OAuth callbacks)
      +-- GlobalNotificationBell
      +-- Routes
          +-- Public: Login, Register, ForgotPassword, Terms, Privacy
          +-- Protected (via ProtectedRoute):
              +-- Dashboard
              +-- ChatPage -> ChatInterface -> N8N webhooks
              +-- Onboarding pages
              +-- Settings pages
              +-- Agent pages (DynamicAgentPage)
          +-- Admin (via AdminRoute):
              +-- AdminDashboard, AdminUsers, AdminSettings, AdminActivity
```

### 4.4 Path Aliases

Configured in both `tsconfig.json` and `vite.config.ts`:

| Alias | Resolves To |
|-------|-------------|
| `@/*` | `./client/*` |
| `@shared/*` | `./shared/*` |

Example usage: `import { supabase } from '@/lib/supabase'`

---

## 5. The Agent System

This is the core domain of Squidgy and the most important subsystem to understand.

### 5.1 Agent Lifecycle

An agent goes from YAML config to live chatbot through this pipeline:

```
1. YAML Config                    agents/configs/{id}.yaml
       |
2. build-agents.js               Reads all YAML, generates TypeScript
       |
3. agents.ts (generated)         Static agent registry for the client
       |
4. Supabase sync                 Upserts to personal_assistant_config table
       |
5. N8N Workflow                  Deployed at n8n.theaiteam.uk
       |
6. Knowledge Base                knowledge_base/agents/{id}/*.md
       |
7. Chat Interface                User talks to agent via ChatInterface component
```

The `build-agents.js` script runs automatically on every `npm run dev` and `npm run build`. It reads all YAML files from `agents/configs/`, generates a TypeScript file, and upserts agent records to Supabase's `personal_assistant_config` table.

### 5.2 Current Agents

| Agent ID | Display Name | Category | State Mgmt | Status |
|----------|-------------|----------|------------|--------|
| `personal_assistant` | Pia - Personal Assistant | GENERAL | No | Active, pinned |
| `brandy` | Brandy - Brand Advisor | MARKETING | Yes (`uses_conversation_state: true`) | Active, pinned |
| `content_repurposer` | Rita - Content Repurposer | CONTENT | No | Disabled |
| `sol_bot` | Sol Bot - Solar Analytics | SOLAR | No | Active |
| `social_media_agent` | Social Media Agent | MARKETING | No | Active |
| `newsletter` | Newsletter Agent | CONTENT | No | Active |
| `agent_builder` | Agent Builder | INTERNAL | No | Internal tool |

Additional YAML files exist for multi-topic variants (`content_repurposer_multi.yaml`, `newsletter_multi.yaml`, `social_media_scheduler.yaml`) that may or may not be active.

### 5.3 Agent YAML Schema

Every agent YAML file follows this structure (example from `brandy.yaml`):

```yaml
agent:
  id: brandy                           # Unique identifier, used as URL slug
  emoji: "..."
  name: "Brandy | Brand Advisor"       # Display name
  category: MARKETING                  # GENERAL, MARKETING, CONTENT, SOLAR, INTERNAL
  description: "..."
  specialization: "Brand Strategist"
  tagline: "Define. Refine. Align."
  avatar: "/Squidgy AI Assistants Avatars/7.png"
  pinned: true                         # Appears in sidebar permanently
  enabled: true                        # Whether agent is active
  uses_conversation_state: true        # CRITICAL: enables state tracking
  initial_message: '...'               # First message shown in chat
  sidebar_greeting: "..."
  capabilities: [...]
  recent_actions: [...]

n8n:
  webhook_url: https://n8n.theaiteam.uk/webhook/brandy

ui_use:
  page_type: single_page
  pages:
    - name: Brandy Dashboard
      path: brandy-dashboard
      order: 1
      validated: true

interface:
  type: chat
  features:
    - text_input
    - file_upload
    - suggestion_buttons

suggestions:
  - "1 - Build from scratch"
  - "2 - Import existing brand docs"

personality:
  tone: friendly
  style: direct
  approach: consultative
```

### 5.4 N8N Webhook Integration

When a user sends a message to an agent, the client calls the agent's N8N webhook URL with this payload:

```json
{
  "body": {
    "user_id": "supabase-uuid",
    "user_mssg": "The user's message",
    "session_id": "chat-session-uuid",
    "request_id": "unique-request-uuid",
    "state": { ... }               // Only if uses_conversation_state: true
  }
}
```

The N8N workflow returns:

```json
{
  "response": "Agent's text reply",
  "Status": "Waiting" | "Ready",   // Capital S -- this is not a typo
  "state": { ... }                 // Updated conversation state
}
```

The `Status` field (capital S) controls whether the agent is still gathering information (`Waiting`) or has finished its task (`Ready`). The frontend reads `state` from the response and sends it back on the next message, enabling multi-turn wizard flows.

### 5.5 Chat Architecture

The chat subsystem is composed of several interacting components:

```
ChatPage.tsx
  +-- ChatInterface.tsx               # Main orchestrator
      +-- ChatHistory.tsx             # Message list with scroll management
      +-- StreamingChatMessage.tsx     # Individual message rendering
      +-- InteractiveMessageButtons   # Suggestion buttons from agent
      +-- QuestionPrompt.tsx          # Structured question display
      +-- ActionsDisplay.tsx          # Action buttons for agent responses
      +-- ContentPreview.tsx          # Rich content previews
      +-- CleanChatInterface.tsx      # Simplified chat variant
      +-- GroupChatInterface.tsx      # Multi-agent chat
      +-- N8nChatInterface.tsx        # Direct N8N integration variant
```

Key services that support chat:

| Service | File | Purpose |
|---------|------|---------|
| Chat History | `chatHistoryService.ts` | Stores/retrieves messages from Supabase. Largest service at ~20KB. |
| Conversation State | `conversationStateService.ts` | Manages multi-turn state for wizard-style agents |
| Chat Session | `chatSessionService.ts` | Session creation and management |
| Content Repurposer Webhook | `contentRepurposerWebhookService.ts` | Calls N8N webhooks for content agents |
| Newsletter Webhook | `newsletterWebhookService.ts` / `newslettersWebhookService.ts` | Newsletter-specific webhook handling |

### 5.6 Dynamic Agent Pages

The `DynamicAgentPage` component (`client/components/DynamicAgentPage.tsx`) renders agent interfaces. It supports two modes:

1. **Chat mode (default):** Renders the standard `ChatInterface` component.
2. **Figma mode:** Dynamically imports a generated React component from `client/pages/agents/`. This was intended for custom UI pages generated from Figma designs but is not widely used.

```typescript
// If pageType === 'figma' && componentPath is set:
const module = await import(`../pages/agents/${fileName}`);
// Otherwise: renders ChatInterface
```

---

## 6. Authentication and Authorization

### 6.1 Supabase Auth Configuration

Authentication uses Supabase's PKCE flow, configured in `client/lib/supabase.ts`:

```typescript
export const supabase = createClient(url, key, {
  auth: {
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: true,
    storage: window.localStorage,
    storageKey: 'supabase.auth.token',
    flowType: 'pkce'
  }
});
```

### 6.2 Route Protection

Two guard components wrap protected routes:

- **`ProtectedRoute`** -- Requires authenticated Supabase session. Redirects to `/login` if no session.
- **`AdminRoute`** -- Extends `ProtectedRoute` with admin role check.

### 6.3 Auth Redirect Handling

The `AuthHandler` component in `App.tsx` intercepts OAuth callback URLs at the root path (`/`). It checks for `code`, `access_token`, and `type` query parameters:

- `type=recovery` redirects to `/reset-password`
- `type=signup` redirects to `/login`
- Generic auth codes redirect to `/login`

An IIFE at the top of `App.tsx` captures email verification parameters from the URL before React mounts:

```typescript
if ((type === 'signup' || type === 'email_change' || code) && type !== 'recovery') {
  sessionStorage.setItem('email_verified', 'true');
}
```

This runs at module load time because Supabase processes and clears URL parameters during initialization.

---

## 7. Build and Deployment

### 7.1 Build Pipeline

The build is split into two Vite configurations:

1. **Client build** (`vite.config.ts`): Produces `dist/spa/` with the React SPA.
2. **Server build** (`vite.config.server.ts`): Produces `dist/server/` with the Express app.

```bash
npm run build
# Equivalent to:
# 1. node scripts/build-agents.js  (YAML -> agents.ts + Supabase sync)
# 2. vite build                    (client -> dist/spa/)
# 3. vite build --config vite.config.server.ts  (server -> dist/server/)
```

### 7.2 Development Server

In development, the Express server is embedded into Vite via a custom plugin (`expressPlugin` in `vite.config.ts`). The plugin intercepts requests starting with `/api` and strips the prefix before passing them to Express:

```typescript
// vite.config.ts -- expressPlugin
if (req.url?.startsWith('/api')) {
  req.url = req.url.slice(4);  // /api/ping -> /ping
  app(req, res, next);
}
```

This means in development, all API calls go through `http://localhost:8080/api/*`.

### 7.3 Vite File System Access

The dev server restricts file system access to three directories:

```typescript
fs: {
  allow: ["./client", "./shared", "./agents"],
  deny: [".env", ".env.*", "*.{crt,pem}", "**/.git/**", "server/**"],
}
```

Server code is explicitly denied from client-side imports.

### 7.4 Vercel Deployment

The `vercel.json` configuration handles:

- **SPA routing:** All non-API routes rewrite to `/index.html`
- **Cache headers:** Static assets (`/assets/*`) get 1-year immutable cache; HTML and API responses get `no-cache`
- **Security headers:** CSP allows framing from GHL domains and `app.squidgy.ai`; XSS protection enabled

```json
{
  "Content-Security-Policy": "frame-ancestors 'self' https://*.gohighlevel.com https://*.leadconnectorhq.com https://app.squidgy.ai"
}
```

---

## 8. Database Schema

### 8.1 Core Tables (Supabase/PostgreSQL)

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `profiles` | User profiles | `user_id`, `email`, `full_name`, `company_id`, `role` |
| `agents` | Agent definitions | `id`, `name`, `category`, `enabled` |
| `personal_assistant_config` | Agent config sync target | `config_type`, `code`, `display_name`, `emoji`, `is_enabled` |
| `brands` | Brandy wizard data | `user_id`, `atmosphere`, `rebellious_edge`, `enemy_statement`, `visual_direction`, `hook_style`, `voice_messaging`, `full_brand_bible` |
| `website_analysis` | Site analysis results | `user_id`, analysis data |
| `business_details` | Company information | `user_id`, business data |
| `solar_setup` | Solar agent config | `user_id`, solar parameters |
| `calendar_setup` | Calendar integration | `user_id`, calendar config |
| `notification_preferences` | User notification settings | `user_id`, preference flags |
| `ghl_subaccounts` | GoHighLevel accounts | `user_id`, GHL account data |
| `chat_history` | Chat message storage | `user_id`, `session_id`, `agent_id`, messages |
| `leads` / `lead_information` | Lead capture | Lead data fields |
| `billing_settings` | Billing configuration | Billing data |

Schema SQL files live in `database/schemas/` and `database/` (root level has some legacy migration files).

### 8.2 Key Relationships

- `profiles.user_id` is the foreign key used across nearly every table
- `personal_assistant_config` uses a composite key of `(config_type, code)` for upserts
- `brands` stores Brandy wizard data per user (one row per user)
- `chat_history` is keyed by `(user_id, session_id, agent_id)`

---

## 9. Client Services Reference

### 9.1 Agent Services

| Service | File | Purpose |
|---------|------|---------|
| Agent Config | `agentConfigService.ts` | Loads agent configuration from YAML-generated data |
| Agent Enablement | `agentEnablementService.ts` | Toggle agents on/off per user |
| Agent Mapping | `agentMappingService.ts` | Maps agent IDs to display names and metadata |
| Optimized Agent | `optimizedAgentService.ts` | Performance-optimized agent data loading |
| YAML Agent Loader | `yamlAgentLoader.ts` | Parses YAML configs at runtime |

### 9.2 Chat Services

| Service | File | Purpose |
|---------|------|---------|
| Chat History | `chatHistoryService.ts` | Message CRUD in Supabase (~20KB, largest service) |
| Chat Session | `chatSessionService.ts` | Session lifecycle management |
| Conversation State | `conversationStateService.ts` | Multi-turn wizard state (singleton pattern) |
| Group Chat | `groupChatService.ts` | Multi-agent chat support |

### 9.3 Content Services

| Service | File | Purpose |
|---------|------|---------|
| Content Repurposer Webhook | `contentRepurposerWebhookService.ts` | N8N webhook for content transformation |
| Content Repurposer Edit | `contentRepurposerEditService.ts` | Edit repurposed content |
| Content Repurposer Parser | `contentRepurposerParser.ts` | Parse content repurposer responses |
| Newsletter Webhook | `newsletterWebhookService.ts` | N8N webhook for newsletter generation |
| Image Service | `imageService.ts` | Image generation and management |

### 9.4 Onboarding Services

| Service | File | Purpose |
|---------|------|---------|
| Onboarding Service | `onboardingService.ts` | Core onboarding flow logic |
| Onboarding Config Loader | `onboardingConfigLoader.ts` | Load onboarding step configurations |
| Onboarding Data Service | `onboardingDataService.ts` | Persist onboarding progress |
| Onboarding Router | `onboardingRouter.ts` | Route logic for onboarding steps |
| Business Flow Loader | `businessFlowLoader.ts` | Multi-step business wizard flows |

### 9.5 Other Services

| Service | File | Purpose |
|---------|------|---------|
| Lead Service | `leadService.ts` | Lead capture and management |
| Referral Service | `referralService.ts` | Referral tracking |
| Referral Flow Loader | `referralFlowLoader.ts` | Referral onboarding flow |
| File Upload | `fileUploadService.ts` | File upload to Supabase Storage |
| Invoice Storage | `invoiceStorageService.ts` | Invoice file management |
| GHL Media | `ghlMediaService.ts` | GoHighLevel media handling |
| Navigation | `navigationService.ts` | Programmatic navigation |
| Supabase Query | `supabaseQueryService.ts` | Shared Supabase query helpers |
| Anonymous Player | `anonymousPlayer.ts` | Anonymous user session handling |

---

## 10. Custom Hooks Reference

| Hook | File | Purpose |
|------|------|---------|
| `useUser` | `useUser.tsx` | Auth state, user profile, Supabase session (provides `UserProvider` context) |
| `useAdmin` | `useAdmin.tsx` | Admin role detection |
| `useCompanyBranding` | `useCompanyBranding.tsx` | Company brand data |
| `useGoogleCalendar` | `useGoogleCalendar.tsx` | Google Calendar integration state |
| `useNavigationService` | `useNavigationService.ts` | Navigation helpers |
| `useThinkingMessage` | `useThinkingMessage.tsx` | "Agent is thinking..." animation state |
| `useWebsiteAnalysis` | `useWebsiteAnalysis.tsx` | Website analysis data and loading state |
| `use-mobile` | `use-mobile.tsx` | Mobile viewport detection |
| `use-toast` | `use-toast.ts` | Toast notification state |

The `hooks/mobile/` subdirectory contains additional mobile-specific hooks.

---

## 11. Server API Routes

### 11.1 Route Map

All routes are prefixed with `/api` in development (Vite strips the prefix before Express sees it).

```
GET  /ping                              -> Health check
GET  /demo                              -> Demo endpoint
POST /website/full-analysis             -> Scrapes and analyzes a URL
POST /website/screenshot                -> Captures page screenshot
POST /website/favicon                   -> Extracts favicon
POST /ghl/create-subaccount-and-user    -> Creates GHL subaccount
/agents/*                               -> Agent CRUD (agentsRouter)
/storage/*                              -> Supabase storage proxy (masks URLs)
/api/google/calendar/*                  -> Google Calendar OAuth flow
```

### 11.2 Storage Proxy

The storage proxy (`server/routes/storage-proxy.ts`) exists to mask Supabase Storage URLs from the client. Instead of exposing `https://xxxxx.supabase.co/storage/v1/...` URLs to the browser, the client calls `/api/storage/...` and the server proxies the request. This prevents leaking the Supabase project URL in client-visible network requests.

---

## 12. UI Component System

### 12.1 Component Architecture

The 52 UI components in `client/components/ui/` follow the shadcn/ui pattern: each is a thin wrapper around a Radix UI primitive that applies Tailwind CSS classes via `class-variance-authority` (CVA) and `tailwind-merge`.

These components are NOT installed from an npm package. They are local source files that were scaffolded from shadcn/ui and can be freely modified. The `components.json` file in the project root configures the shadcn/ui CLI for adding new components.

Key utilities used across all UI components:

```typescript
import { cn } from "@/lib/utils"  // clsx + tailwind-merge
import { cva, type VariantProps } from "class-variance-authority"
```

### 12.2 Radix Primitives in Use

Accordion, AlertDialog, AspectRatio, Avatar, Checkbox, Collapsible, ContextMenu, Dialog, DropdownMenu, HoverCard, Label, Menubar, NavigationMenu, Popover, Progress, RadioGroup, ScrollArea, Select, Separator, Slider, Slot, Switch, Tabs, Toast, Toggle, ToggleGroup, Tooltip.

### 12.3 Additional UI Libraries

- **cmdk** (1.1.1): Command palette / search
- **embla-carousel-react** (8.6.0): Carousel component
- **input-otp** (1.4.2): OTP code input
- **react-day-picker** (9.8.1): Date picker
- **react-resizable-panels** (3.0.4): Resizable split panes
- **sonner** (1.7.4): Toast notifications (used alongside Radix Toast)
- **vaul** (1.1.2): Drawer component
- **canvas-confetti** (1.9.4): Celebration animations

---

## 13. Tribal Knowledge and Gotchas

This section contains hard-won knowledge from production incidents, debugging sessions, and integration quirks. Read this section carefully before making changes. If you are an AI coding agent, treat every item here as a constraint.

### 13.1 N8N Integration Rules

**RULE: N8N webhook URLs MUST use `/webhook/` not `/webhook-test/` in YAML configs.**

The `/webhook-test/` path only works when the N8N workflow editor has the workflow open in test mode. In production, only `/webhook/` works. If an agent stops responding, check the YAML config first.

```yaml
# CORRECT
n8n:
  webhook_url: https://n8n.theaiteam.uk/webhook/brandy

# WRONG - will silently fail in production
n8n:
  webhook_url: https://n8n.theaiteam.uk/webhook-test/brandy
```

**RULE: Never modify a working N8N workflow directly.** Duplicate the workflow first, make changes on the duplicate, test it, then switch the webhook URL in the YAML config. The N8N API cannot set credentials, activate workflows, or manage tags -- those are UI-only operations.

**RULE: Test N8N webhooks with curl and check response body size.** A response body of 0 bytes means the Respond node in the workflow never fired -- the workflow broke before reaching it.

```bash
N8N_KEY=$(cat /Users/sethward/GIT/Squidgy/.n8n-api-key | tr -d '\n')
curl -s -X POST https://n8n.theaiteam.uk/webhook/brandy \
  -H "Content-Type: application/json" \
  -d '{"body":{"user_id":"test","user_mssg":"hello","session_id":"test","request_id":"test"}}' \
  | python3 -m json.tool
```

### 13.2 Conversation State

**RULE: `uses_conversation_state: true` MUST be set in the agent YAML for the frontend to send state.**

Without this flag, the frontend will not include the `state` field in webhook payloads, and the N8N workflow will receive `undefined` for state. The agent will appear to "forget" previous wizard steps.

The frontend reads `state` (lowercase) from the request body at `$input.first().json.body.state`, not `conversation_state`.

The N8N output parser schema for stateful agents is:

```json
{
  "response": "string",
  "Status": "Waiting | Ready",
  "state": "object"
}
```

Note the capital `S` in `Status`. This is intentional and must not be changed to lowercase.

### 13.3 Supabase Client Dummy Fallback

The Supabase client in `client/lib/supabase.ts` falls back to dummy values if environment variables are not set:

```typescript
const finalUrl = supabaseUrl || 'https://dummy.supabase.co';
const finalKey = supabaseAnonKey || 'dummy-key';
```

This prevents the app from crashing during local development when `.env` is not configured, but it means **all Supabase operations will fail silently**. If you are debugging and nothing is saving to the database, check that your `.env` file has real Supabase credentials.

### 13.4 Build Pipeline Ordering

The agent build step (`build-agents.js`) runs before every Vite build. This is enforced in `package.json`:

```json
"dev": "node scripts/build-agents.js && vite",
"build:client": "node scripts/build-agents.js && vite build"
```

If you add a new agent YAML but skip the build step, the client will not see it. The `dev:watch` command (`vite` only) skips the agent build, so new YAML changes will not appear until you restart `npm run dev`.

### 13.5 TypeScript is Lenient

The project runs with `strict: false`, `noImplicitAny: false`, and `strictNullChecks: false`. This means:

- `any` types are everywhere and the compiler will not warn about them
- Null/undefined access will not be caught at compile time
- `npm run typecheck` will pass even with type-unsafe code

If you are adding new code, aim for stricter typing than the existing codebase, but do not turn on strict mode project-wide (it will produce hundreds of errors).

### 13.6 Express 5 (Not Express 4)

The server uses Express 5.1.0, which has breaking changes from Express 4:

- `req.query` returns a plain object (not `qs` parsed)
- Route parameter regex syntax changed
- Error handling middleware has different semantics
- Some Express 4 middleware may not be compatible

### 13.7 Path Alias Gotcha

The `@/*` alias resolves to `./client/*` and `@shared/*` resolves to `./shared/*`. These are configured in BOTH `tsconfig.json` (for TypeScript) and `vite.config.ts` (for the bundler). If you add a new alias, you must update both files or imports will work in the IDE but fail at build time (or vice versa).

### 13.8 Agent Avatar Paths

Agent avatars reference paths like `/Squidgy AI Assistants Avatars/1.png`. These static files must exist in the `public/` directory. The path includes spaces, which is unusual and can cause issues with some tooling.

### 13.9 Multiple Onboarding Flows

There are at least three onboarding implementations in the codebase:

1. **Original onboarding** -- `client/pages/onboarding/` (7 pages, route-based wizard)
2. **ComprehensiveOnboarding** -- `client/components/onboarding/ComprehensiveOnboarding.tsx` (8-step single-component wizard)
3. **New onboarding** -- `client/pages/new_onboarding/` (latest iteration)

All three are still routed in `App.tsx`. The current active flow should be verified before making changes.

### 13.10 Root Directory Clutter

The `squidgy_updated_ui/` root contains many legacy files that should not be there: Python scripts (`fix_brandy_*.py`, `deploy_brandy_*.py`, `activate_brandy_*.py`), loose SQL files, loose `.tsx` files (`ChatPage.tsx`, `WelcomeModal.tsx`), and various debugging/deployment docs. These are artifacts from rapid development. Do not treat them as authoritative. The canonical locations for each file type are defined in the project's `CLAUDE.md`.

### 13.11 CSP and Embedding

The Content Security Policy in `vercel.json` allows the app to be embedded in iframes from GoHighLevel domains. If the app fails to load in a GHL iframe, check the CSP `frame-ancestors` directive. The current allowed origins are:

- `'self'`
- `https://*.gohighlevel.com`
- `https://*.leadconnectorhq.com`
- `https://app.squidgy.ai`

### 13.12 Two Toast Systems

The app uses two toast notification systems simultaneously:

1. **Radix UI Toast** (`client/components/ui/toaster.tsx` + `use-toast.ts` hook)
2. **Sonner** (`client/components/ui/sonner.tsx`)

Both are mounted in `App.tsx`. Use Sonner for new code (simpler API), but be aware that existing code may use either.

### 13.13 Supabase Query Patterns

When querying Supabase in N8N workflows (not in the frontend code, but relevant for the agent system):

- Use `getAll` not `get` -- `get` crashes on no matching row
- Always use `condition: "eq"` for filtering
- Always set `alwaysOutputData: true` to prevent workflow stalls on empty results

---

## 14. Known Technical Debt

| Area | Issue | Impact |
|------|-------|--------|
| Root directory | 30+ legacy scripts and docs dumped in project root | Confusing for new developers |
| TypeScript strictness | `strict: false` across the whole project | Silent type errors, runtime crashes |
| Test coverage | Minimal Vitest coverage | Regressions caught in production |
| Duplicate onboarding | Three separate onboarding implementations | Maintenance burden, unclear which is active |
| Dual toast systems | Radix Toast + Sonner both mounted | Inconsistent notification UX |
| Large services | `chatHistoryService.ts` is ~20KB | Hard to maintain, needs splitting |
| Dead code | Unused Python scripts, old workflow files | Noise in the codebase |
| `shared/api.ts` | Contains only `DemoResponse` interface | Underutilized shared types layer |

---

## 15. Adding a New Agent (Step-by-Step)

This is the most common modification task. Follow these steps exactly:

1. **Create YAML config** at `agents/configs/{agent_id}.yaml` following the schema in Section 5.3. Use an existing YAML file as a template.

2. **Create knowledge base** directory at `knowledge_base/agents/{agent_id}/` with at minimum an `instructions.md` file.

3. **Set up N8N workflow** following the 10-node pattern: Webhook, Supabase (getAll), Code, AI Agent (with OpenRouter + Memory + Output Parser), If Ready, 2x Respond nodes. Deploy via the N8N UI (the API cannot set credentials or activate).

4. **Update Pia's system prompt** at `knowledge_base/agents/personal_assistant/system_prompt.md` in three locations:
   - Agent Relevance Rules table (~line 215)
   - Agent IDs table (~line 262)
   - Routing table (~line 284)

5. **Run the build pipeline:**
   ```bash
   cd squidgy_updated_ui
   node scripts/build-agents.js
   ```

6. **Test the webhook:**
   ```bash
   curl -s -X POST https://n8n.theaiteam.uk/webhook/{agent_id} \
     -H "Content-Type: application/json" \
     -d '{"body":{"user_id":"test","user_mssg":"hello","session_id":"test","request_id":"test"}}' \
     | python3 -m json.tool
   ```

7. **Verify** the agent appears in the sidebar and responds to messages.

---

## 16. Common Modification Patterns

### 16.1 Adding a New Page

1. Create the page component in `client/pages/`
2. Import it in `client/App.tsx`
3. Add a `<Route>` element inside the `<Routes>` block (above the catch-all `*` route)
4. Wrap with `<ProtectedRoute>` if authentication is required

### 16.2 Adding a New UI Component

The project uses the shadcn/ui CLI pattern. To add a new Radix-based component:

```bash
npx shadcn@latest add [component-name]
```

Or manually create a file in `client/components/ui/` following the existing pattern (CVA variants + Radix primitive + `cn()` utility).

### 16.3 Adding a New Server Route

1. Create a route handler in `server/routes/`
2. Import and mount it in `server/index.ts`
3. Add shared types to `shared/api.ts` if the route has a typed response
4. Call from the client via Axios to `/api/[route-path]`

### 16.4 Adding a New Client Service

1. Create the service file in `client/services/`
2. Import the Supabase client from `@/lib/supabase` if database access is needed
3. Export functions or a singleton class
4. Use from components via direct import (no dependency injection)

---

## 17. Deployment Checklist

Before deploying to production:

- [ ] All agent YAML configs use `/webhook/` URLs (not `/webhook-test/`)
- [ ] `build-agents.js` ran successfully (check for Supabase sync errors)
- [ ] `npm run build` completes without errors
- [ ] `npm run typecheck` passes
- [ ] N8N workflows are activated (not just saved) in the N8N UI
- [ ] Environment variables are set in Vercel dashboard
- [ ] CSP headers in `vercel.json` include any new embedding domains

---

*This document reflects the state of the codebase as of 2026-02-22. Update the Change Log table when making significant architectural changes.*
