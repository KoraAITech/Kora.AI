# KORA AI — FINALIZED MVP SPECIFICATION & MASTER PROMPTS
## Single Source of Truth for All Development Decisions
### Version 1.0 — Locked for MVP Development Phase

---

> **IMPORTANT:** This document is the single source of truth for the entire
> Kora AI MVP. Every decision recorded here was explicitly confirmed by the
> founding team. Do not deviate from these specs during MVP development.
> All future changes must be versioned and noted at the bottom of this file.
> When asking an AI for help, paste the relevant Master Prompt from Section 2
> at the top of your conversation BEFORE your question.

---

# SECTION 1 — FINALIZED MVP SPECIFICATION

---

## 1.1 PRODUCT IDENTITY

| Field | Value |
|---|---|
| Product Name | Kora AI |
| Domain | TBD (decided before launch) |
| Phase | MVP — Phase 1 |
| Target Launch | 4–6 weeks from development start |
| Billing in MVP | None — completely free for all users |
| Billing Phase | Phase 2 (after MVP validated) |

---

## 1.2 TEAM

| Field | Value |
|---|---|
| Team Size | 3 developers |
| Experience Level | Junior / vibe coders using Claude AI |
| Working Style | Fully collaborative — all 3 work on all areas |
| AI Tooling | Claude AI as primary development assistant |
| Dev Environment | Docker (local) for entire MVP development phase |
| Deployment Decision | AWS deployment stack decided post-MVP (deferred) |

**Implication for code style:** Keep code explicit and readable over clever.
Avoid over-engineering. Every file should be understandable by a junior
developer without extensive context. Prefer simple patterns over advanced ones.

---

## 1.3 WHAT KORA AI DOES (MVP SCOPE)

Kora AI is a SaaS platform where any business or developer can:

1. Sign up for a free account on the Kora AI website
2. Upload a single PDF document (their product manual, FAQ, knowledge base, etc.)
3. Kora AI trains an AI chatbot on that document using RAG (Retrieval-Augmented Generation)
4. The user tests the chatbot in a built-in chat UI inside their dashboard
5. The user gets an API key and embed code to deploy the chatbot on their own website
6. Their website visitors can then chat with the bot, which answers from the PDF

**What Kora AI is NOT in MVP:**
- Not a multi-document chatbot (one PDF per chatbot)
- Not a paid service (free in MVP)
- Not a multi-chatbot platform (one active chatbot per user, with version history)
- Not supporting web search or live data (document-only in MVP)
- Not multilingual (English only)

---

## 1.4 USER ACCOUNTS & AUTHENTICATION

| Decision | Choice | Notes |
|---|---|---|
| Login methods | Email+Password AND Google OAuth | Both available from day 1 |
| Email verification | Required before dashboard access | User cannot use the app without verifying |
| Password reset | Yes — email link flow | |
| Social login library | django-allauth | Google OAuth via allauth |
| JWT setup | Access token: 15 min / Refresh: 7 days | Refresh token rotates on each use |
| Token storage | Access token in memory (Zustand) | Refresh token in HttpOnly cookie |
| Session on page refresh | Call /auth/token/refresh/ on app load | Restore session silently from cookie |
| 2FA | Not in MVP | Phase 2 |
| Profile fields | Full name, email, avatar (optional) | Minimal profile for MVP |

**Email verification flow:**
- User registers → immediately redirected to "Check your email" page
- Cannot access dashboard until email is verified
- Verification link expires in 24 hours
- User can request resend from the verification page

---

## 1.5 CHATBOT SPECIFICATION

### One Chatbot Per User with Version History

| Decision | Choice |
|---|---|
| Max chatbots per user | 1 active chatbot at a time |
| Max PDF size | 10 MB hard limit (enforced client AND server) |
| PDF language | English only |
| Re-upload behavior | New upload creates a new version, old archived |
| Version visibility | Previous PDFs listed in dashboard (reference only) |
| Active version | Only the latest upload is active and deployable |
| Version limit | Keep last 3 versions, auto-delete older ones |
| Version switching | User can view old versions but CANNOT roll back in MVP |

**Note on version switching:** Rollback to a previous version is a Phase 2 feature.
In MVP, the version history is read-only — users can see what they uploaded before
but the active chatbot is always the most recent upload.

### Chatbot Customization

| Field | MVP Behaviour |
|---|---|
| Bot name | User sets a custom name (e.g., "Aria", "Support Bot") |
| Greeting message | User writes a custom opening message |
| Personality / system prompt | User writes custom instructions for the bot's persona |
| Avatar | Not in MVP (generic robot icon) |
| Colors | User picks primary color for widget (hex or preset palette) |

**System prompt structure (built by FastAPI):**
```
[User's custom personality instructions]

You are a helpful assistant for [Bot Name]. Answer questions based on the
provided context from the document. If the answer is not found in the
document context, use your general knowledge to provide a helpful response,
but clearly indicate that the answer is not from the provided document.
Always be polite and concise.

[Retrieved document chunks as context]
```

### LLM Configuration

| Decision | Choice |
|---|---|
| Primary LLM | Groq — Llama 3.3 70B (free tier, fast inference) |
| Fallback LLM | Google Gemini 1.5 Flash (automatic if Groq is down/rate-limited) |
| Fallback trigger | Groq API error OR Groq rate limit hit |
| User-selectable model | No — handled automatically, invisible to user in MVP |
| Embedding model | sentence-transformers (local, free, no API cost) — all-MiniLM-L6-v2 |
| Embedding dimensions | 384 (MiniLM) |
| Chunk size | 500 tokens |
| Chunk overlap | 50 tokens |
| Top-K retrieval | 5 chunks per query |
| Similarity threshold | 0.3 minimum score (below this, treat as no context found) |
| Max response tokens | 600 tokens |
| Fallback behavior | If no relevant chunks found: answer from general LLM knowledge, clearly noting it's not from the document |

**Why sentence-transformers for embeddings?**
Free, runs locally in Docker, no API cost, good quality for English documents.
Embedding generation happens inside the FastAPI container.

### Training Pipeline

| Step | Detail |
|---|---|
| 1. User uploads PDF | Django receives, validates (size + MIME), stores in S3/volume |
| 2. Django calls FastAPI | POST /ingest/ — fire and forget, returns 202 immediately |
| 3. Chatbot status → "processing" | Stored in DB, frontend polls every 3 seconds |
| 4. FastAPI ingests | Download PDF → extract text → chunk → embed → store in pgvector |
| 5. FastAPI webhooks Django | POST /webhook/train-complete/ with status + chunk count |
| 6. Status → "ready" | Frontend polling detects "ready", shows "Start Chatting" button |
| 7. Email sent | "Your chatbot is ready!" notification email sent to user |

**Training failure handling:**
- If FastAPI fails at any step, it webhooks Django with `status: "failed"` and an error message
- Django updates chatbot status to "failed"
- Frontend shows error state with a "Try Again" button (triggers re-processing of same PDF)
- Error is logged for the team to investigate

---

## 1.6 INTERNAL CHAT (DASHBOARD TEST UI)

| Decision | Choice |
|---|---|
| Purpose | User tests their chatbot before deploying it |
| Chat history saved | Yes — saved per chatbot version, user can review |
| History visible | Full conversation history in a sidebar list |
| Sessions | Each "New Chat" creates a new session |
| History retention | Kept indefinitely in MVP (cleanup in Phase 2) |
| Streaming | Yes — tokens stream in real time via SSE |
| Max message length | 1000 characters per message |
| Context window | Last 6 messages sent as history to LLM |

---

## 1.7 EMBED & API SPECIFICATION

### How Customers Deploy the Chatbot

Kora AI provides two embed options:

**Option 1 — Script Tag (for any website)**
```html
<script
  src="https://cdn.koraai.com/widget.js"
  data-chatbot-id="YOUR_CHATBOT_ID"
  data-api-key="YOUR_API_KEY"
  defer
></script>
```
Paste before `</body>`. Widget auto-initializes as a floating bubble.

**Option 2 — React Component (for React/Next.js apps)**
```bash
npm install @kora-ai/widget
```
```jsx
import { KoraChatWidget } from '@kora-ai/widget'

<KoraChatWidget chatbotId="YOUR_CHATBOT_ID" apiKey="YOUR_API_KEY" />
```

Distribution: npm package published to npm registry.
Copy-paste component code also available in dashboard as fallback.

### API Key Rules

| Decision | Choice |
|---|---|
| Key scope | One API key per chatbot (not per user) |
| Key format | `kora_live_[8-char-prefix]_[random-secret]` |
| Key storage | Only SHA-256 hash stored in DB. Full key shown once. |
| Key rotation | User can rotate key. Old key immediately invalidated. |
| Multiple keys | Not in MVP — one active key per chatbot |
| Key reveal | Masked in UI (show prefix + last 4). "Reveal" toggle. |

### Widget Appearance & Customization

| Setting | User Controls |
|---|---|
| Bot name | Displayed in widget header |
| Primary color | Widget bubble + header color (hex input or 6 presets) |
| Greeting message | First message shown when widget opens |
| Widget position | Fixed: bottom-right (not configurable in MVP) |
| Widget size | Fixed: standard (not configurable in MVP) |

### Embed Chat Behavior

| Decision | Choice |
|---|---|
| Conversation persistence | Server-side session with a session ID stored in localStorage |
| Session ID | UUID generated by widget on first load, stored in localStorage |
| History per session | Maintained in FastAPI Redis cache for 24 hours |
| New session trigger | User clears localStorage OR 24-hour session expiry |
| Rate limiting | 20 messages per minute per API key |
| CORS | Open (allow all origins) — widget called from any domain |
| Max message length | 1000 characters |

---

## 1.8 PUBLIC MARKETING WEBSITE PAGES

All 6 pages required in MVP:

| Page | Path | Purpose |
|---|---|---|
| Homepage | / | Hero, features overview, how it works, CTA to sign up |
| Pricing | /pricing | Free plan details, Phase 2 teaser, FAQ |
| How it Works | /how-it-works | Step-by-step visual walkthrough + demo GIF/video |
| Contact / Support | /contact | Support form + email, FAQ accordion |
| Documentation | /docs | API reference, embed guide, quickstart |
| Blog | /blog | Articles (at least 1 launch post for SEO) |

---

## 1.9 TRANSACTIONAL EMAILS

| Email | Trigger | Required in MVP |
|---|---|---|
| Welcome | After email verification confirmed | ✅ Yes |
| Email Verification | After registration | ✅ Yes |
| Password Reset | User requests reset | ✅ Yes |
| Chatbot Ready | Training completes successfully | ✅ Yes |
| Training Failed | Training fails | ✅ Yes (include retry link) |

**Email provider:** AWS SES (consistent with AWS infrastructure)
**Sender:** noreply@koraai.com (or TBD domain)
**Library:** django-anymail with SES backend

---

## 1.10 ADMIN PANEL

| Decision | Choice |
|---|---|
| Type | Django Admin (built-in, customized) |
| Access | Staff users only (is_staff = True) |
| Capabilities | View users, view chatbots, view API keys, delete users, ban users |
| Analytics | None in MVP — view raw model data only |
| Custom dashboard | No — Phase 2 |

---

## 1.11 USAGE LIMITS (MVP)

| Limit | Value | Enforcement |
|---|---|---|
| PDF file size | 10 MB | Client-side (before upload) + server-side (reject if exceeded) |
| Embed messages per minute | 20 per API key | Redis sliding window counter in FastAPI |
| Embed messages per month | None in MVP | Monitor and add in Phase 2 |
| Beta user cap | None | Open signups |
| Chatbots per user | 1 active | Enforced at Django model level |
| PDF versions kept | Last 3 | Older auto-deleted via scheduled Celery task |

---

## 1.12 LOCAL DEVELOPMENT ENVIRONMENT

Everything runs in Docker. One command starts the entire stack.

```
docker-compose up
```

**Services in docker-compose.yml:**

| Service | Image | Port | Purpose |
|---|---|---|---|
| nextjs | Custom Dockerfile.dev | 3000 | Next.js frontend (hot reload) |
| django | Custom Dockerfile.dev | 8000 | Django backend (hot reload) |
| fastapi | Custom Dockerfile.dev | 8001 | FastAPI RAG service (hot reload) |
| postgres | postgres:16 | 5432 | Database (Django + FastAPI share) |
| redis | redis:7-alpine | 6379 | Celery broker + session cache |
| celery | Same as Django | — | Background task worker |
| pgadmin | dpage/pgadmin4 | 5050 | Database GUI (dev only) |

**File storage in development:**
Use local Docker volume (not S3) for PDF storage.
S3 only in staging/production.
Django settings auto-detect environment and switch storage backend.

**AWS Deployment:** Decision deferred. Will be finalized after MVP is
functionally complete and tested in Docker.

---

## 1.13 WHAT IS EXPLICITLY OUT OF SCOPE FOR MVP

The following are confirmed NOT being built in MVP:

| Feature | Phase |
|---|---|
| Paid plans / Stripe billing | Phase 2 |
| Multiple chatbots per user | Phase 2 |
| Rollback to previous chatbot version | Phase 2 |
| Web search / live data in responses | Phase 2 |
| Chatbot analytics dashboard | Phase 2 |
| Team/organization accounts | Phase 2 |
| Multilingual PDF support | Phase 2 |
| User-selectable LLM model | Phase 2 |
| White-label widget | Phase 2 |
| Chatbot trained on website URL | Phase 2 |
| Multiple PDF upload per chatbot | Phase 2 |
| Chatbot conversation analytics | Phase 2 |
| Webhook notifications to customer's server | Phase 2 |
| 2FA / MFA | Phase 2 |
| API rate limit dashboard for users | Phase 2 |

---

# SECTION 2 — MODULE MASTER PROMPTS

---

> **HOW TO USE:** Copy the entire prompt block for the relevant module.
> Paste it at the TOP of a new AI conversation. Then write your question
> below it. Never skip the master prompt — it prevents the AI from giving
> generic answers that don't fit Kora AI's specific architecture.

---

## MASTER PROMPT 1 — FRONTEND (Next.js)

```
MASTER CONTEXT — KORA AI FRONTEND

Project: Kora AI — SaaS RAG chatbot platform.
I need help with the Next.js frontend. Read all context below before answering.
Do NOT give generic answers. Everything must fit this exact project.

━━━ PRODUCT SUMMARY ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Kora AI lets users sign up, upload a PDF, train an AI chatbot on it, test
it in a dashboard chat UI, then embed it on their own website via a script
tag or React npm package. MVP is free. One chatbot per user (with version
history — last 3 PDFs kept, only latest is active).

━━━ TECH STACK ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Framework:        Next.js 14 — App Router (NOT Pages Router)
Language:         TypeScript — strict mode
Styling:          Tailwind CSS + shadcn/ui
State:            Zustand (global) + TanStack Query / React Query (server state)
Forms:            React Hook Form + Zod validation
HTTP:             Axios with shared instance (auto JWT header + auto-refresh on 401)
Auth:             JWT from Django. Access token in Zustand (memory only, never
                  localStorage). Refresh token in HttpOnly cookie set by Django.
Real-time:        SSE (Server-Sent Events) for streaming chat responses
                  Polling (every 3s) for chatbot training status
Icons:            Lucide React
Testing:          Jest + React Testing Library (unit), Playwright (E2E)
Dev environment:  Docker (hot reload, port 3000)

━━━ APP ROUTER STRUCTURE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

src/
├── app/
│   ├── (marketing)/              ← Public pages, no auth
│   │   ├── page.tsx              ← Homepage
│   │   ├── pricing/page.tsx
│   │   ├── how-it-works/page.tsx
│   │   ├── contact/page.tsx
│   │   ├── docs/page.tsx
│   │   └── blog/
│   ├── (auth)/                   ← Auth pages, redirect to dashboard if logged in
│   │   ├── login/page.tsx
│   │   ├── register/page.tsx
│   │   ├── verify-email/page.tsx ← User lands here after registration
│   │   └── forgot-password/page.tsx
│   ├── (dashboard)/              ← Protected: redirect to /login if no session
│   │   ├── layout.tsx            ← Sidebar + header shell
│   │   ├── overview/page.tsx
│   │   └── chatbot/
│   │       ├── page.tsx          ← Chatbot home: shows upload state or active bot
│   │       ├── train/page.tsx    ← PDF upload + training progress
│   │       ├── test/page.tsx     ← Internal chat UI for testing
│   │       ├── deploy/page.tsx   ← API key + script tag + React snippet
│   │       ├── customize/page.tsx← Bot name, greeting, colors, system prompt
│   │       └── history/page.tsx  ← Previous PDF versions (read-only)
│   └── api/                      ← Minimal Next.js API routes if needed
├── components/
│   ├── ui/                       ← shadcn/ui base components
│   ├── layout/                   ← Sidebar, Header, MobileMenu
│   ├── chatbot/
│   │   ├── PDFUploader.tsx       ← Drag-drop, 10MB validation, progress bar
│   │   ├── TrainingStatus.tsx    ← Polls /chatbot/status/ every 3s
│   │   ├── ChatWindow.tsx        ← SSE streaming chat UI
│   │   ├── ChatMessage.tsx       ← Single message bubble (user/assistant)
│   │   ├── APIKeyCard.tsx        ← Masked key, reveal, copy, rotate
│   │   ├── ScriptSnippet.tsx     ← Syntax-highlighted script tag
│   │   ├── ReactSnippet.tsx      ← npm install + JSX usage snippet
│   │   └── CustomizeForm.tsx     ← Bot name, greeting, color, system prompt
│   └── shared/
├── hooks/
│   ├── useAuth.ts
│   ├── useChatbot.ts
│   ├── useChat.ts                ← SSE streaming hook
│   └── useTrainingStatus.ts     ← Polling hook
├── lib/
│   ├── api/
│   │   ├── axios.ts              ← Axios instance with JWT interceptor
│   │   ├── auth.ts               ← Auth API calls
│   │   └── chatbot.ts            ← Chatbot API calls
│   ├── auth/
│   │   └── tokens.ts             ← Token management utilities
│   └── utils/
├── store/
│   ├── authStore.ts              ← User + access token in Zustand
│   └── chatbotStore.ts           ← Current chatbot state
├── types/
│   ├── auth.ts
│   └── chatbot.ts
└── middleware.ts                 ← Protects /dashboard/* routes

━━━ AUTHENTICATION RULES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Access token ONLY in Zustand (memory). NEVER in localStorage / sessionStorage.
- Refresh token is in an HttpOnly cookie — JavaScript cannot read it.
- On every app load: call POST /api/v1/auth/token/refresh/ silently.
  If it succeeds: store access token in Zustand, user is logged in.
  If it fails: redirect to /login.
- middleware.ts uses a "session exists" cookie (not the JWT) to gate
  /dashboard/* routes. The real auth happens client-side on load.
- After login: redirect to /dashboard/overview
- After register: redirect to /verify-email (cannot use dashboard until verified)
- After email verification: redirect to /dashboard/overview

━━━ DJANGO API ENDPOINTS FRONTEND CALLS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Base URL: http://localhost:8000/api/v1 (dev) / https://api.koraai.com/api/v1 (prod)

Auth:
  POST  /auth/register/            { email, password, full_name }
  POST  /auth/login/               { email, password }
  POST  /auth/logout/
  POST  /auth/token/refresh/       (uses HttpOnly cookie)
  POST  /auth/google/              { code } — Google OAuth
  POST  /auth/verify-email/        { token }
  POST  /auth/resend-verification/
  POST  /auth/password/reset/      { email }
  POST  /auth/password/reset/confirm/ { token, password }
  GET   /auth/me/
  PATCH /auth/me/

Chatbot:
  GET   /chatbot/                  Get user's current chatbot info
  POST  /chatbot/upload/           Multipart PDF upload
  GET   /chatbot/status/           Poll training status
  POST  /chatbot/chat/             Internal test chat (SSE stream)
  PATCH /chatbot/customize/        Update name, greeting, color, system prompt
  GET   /chatbot/history/          List previous PDF versions
  GET   /chatbot/api-key/          Get current API key (masked)
  POST  /chatbot/api-key/rotate/   Rotate API key

━━━ KEY COMPONENT BEHAVIOURS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PDFUploader:
  - Drag and drop zone + click to browse
  - Client-side validation BEFORE upload: must be PDF, max 10 MB
  - Show file name + size after selection
  - Show upload progress bar (Axios onUploadProgress)
  - On success → redirect to /dashboard/chatbot/train (shows training status)

TrainingStatus:
  - Polls GET /chatbot/status/ every 3 seconds
  - States: uploading → processing → ready / failed
  - Shows animated progress indicator during processing
  - On "ready" → show success state + "Start Testing" button + trigger email sent note
  - On "failed" → show error message + "Try Again" button

ChatWindow (internal test):
  - SSE connection to POST /chatbot/chat/
  - Messages stream token by token with blinking cursor
  - Chat history loaded from DB on page load (saved sessions)
  - "New Conversation" button starts a fresh session
  - Previous sessions listed in a sidebar
  - Max input: 1000 characters (show character counter)

APIKeyCard:
  - Key shown as: kora_live_Ab3xYz12_••••••••••••••••••••••••••••1234
  - "Reveal" button shows full key for 30 seconds then re-masks
  - "Copy" button copies to clipboard (show checkmark confirmation)
  - "Rotate Key" button opens a confirmation modal:
    "Your current key will stop working immediately. Continue?"

ScriptSnippet / ReactSnippet:
  - Syntax highlighted code blocks (use shiki or highlight.js)
  - One-click copy button on each
  - Tab switcher: "Script Tag" | "React Component"
  - React tab shows: npm install command + JSX usage example

CustomizeForm:
  - Bot name: text input, required, max 50 chars
  - Greeting message: textarea, max 200 chars
  - System prompt / personality: textarea, max 1000 chars,
    placeholder: "e.g. You are a friendly support agent for..."
  - Primary color: 6 preset color swatches + hex input
  - Live preview panel showing widget with current settings
  - Auto-save on blur (no save button needed — PATCH on field change)

━━━ RULES — DO NOT SUGGEST ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✗ Pages Router — we use App Router
✗ Redux — we use Zustand
✗ NextAuth — Django handles all auth
✗ localStorage for tokens — security risk
✗ plain fetch — we use Axios
✗ WebSockets for chat — we use SSE
✗ Mock data in components — all data from real API via React Query

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

## MASTER PROMPT 2 — BACKEND (Django)

```
MASTER CONTEXT — KORA AI BACKEND (DJANGO)

Project: Kora AI — SaaS RAG chatbot platform.
I need help with the Django backend. Read all context below before answering.
Do NOT give generic answers. Everything must fit this exact project.

━━━ WHAT DJANGO DOES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Django handles: user auth (JWT + Google OAuth), email flows, chatbot metadata
(name, status, versions, customization), PDF receipt + forwarding to FastAPI,
API key generation, internal chat proxying, Celery background tasks, and
the Django admin panel. Django does NOT process PDFs, generate embeddings,
or run inference — that is all FastAPI.

━━━ TECH STACK ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Language:        Python 3.12
Framework:       Django 5 + Django REST Framework 3.15
Auth:            djangorestframework-simplejwt (JWT)
                 django-allauth (Google OAuth)
                 Access token: 15 min | Refresh token: 7 days (rotated, HttpOnly cookie)
Database:        PostgreSQL 16 — Django ORM via psycopg2-binary
Cache/Queue:     Redis 7 — django-redis (cache) + Celery 5 (tasks)
Storage:         Local Docker volume in dev | AWS S3 in production
                 django-storages + boto3 (auto-switch via DJANGO_ENV setting)
HTTP to FastAPI: httpx (async HTTP client)
Email:           django-anymail with AWS SES backend
API Docs:        drf-spectacular (Swagger at /api/docs/)
Env vars:        django-environ
CORS:            django-cors-headers
Dev port:        8000

━━━ DJANGO APPS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

config/
  settings/
    base.py        ← shared settings
    local.py       ← Docker dev (local file storage, console email)
    production.py  ← S3 storage, SES email, hardened security

apps/
  users/           ← User model, auth endpoints, Google OAuth, email tasks
  chatbot/         ← Chatbot model, upload, status, chat proxy, API keys, versions
  core/            ← Shared: exceptions, pagination, response renderer, middleware

━━━ DATABASE MODELS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User:
  id (UUID PK), email (unique), password_hash, full_name,
  avatar (optional), is_active, is_staff, is_email_verified,
  auth_provider (email/google), google_id (nullable),
  created_at, updated_at, deleted_at (soft delete)

Chatbot:
  id (UUID PK), user (FK→User, unique — one per user constraint via partial index),
  name (custom bot name), greeting_message, system_prompt, primary_color,
  is_active (bool — True = currently deployed version),
  status (uploading/processing/ready/failed),
  pdf_filename, pdf_s3_key (or local path in dev), pdf_size_bytes,
  version_number (integer, auto-increment per user),
  trained_at, total_pages, chunk_count, error_message,
  created_at, updated_at, deleted_at (soft delete for version archiving)

  CONSTRAINT: Only 1 non-deleted active chatbot per user
  (partial unique index: UNIQUE(user_id) WHERE deleted_at IS NULL AND is_active=TRUE)

  VERSION LOGIC: When user uploads new PDF:
    1. Set old chatbot is_active=False (archived, NOT deleted)
    2. Create new Chatbot record with incremented version_number, is_active=True
    3. Keep last 3 versions. Auto-delete (soft) versions beyond 3.

APIKey:
  id (UUID PK), chatbot (FK→Chatbot, unique — one key per chatbot),
  user (FK→User), key_prefix (8 chars, plaintext for lookup),
  key_hash (SHA-256 hex, 64 chars), is_active,
  created_at, last_used_at, usage_count (bigint)

ChatSession:  (internal test UI only)
  id (UUID PK), user (FK), chatbot (FK), created_at

ChatMessage:  (internal test UI only)
  id (UUID PK), session (FK→ChatSession), role (user/assistant),
  content (text), created_at

━━━ ALL API ENDPOINTS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

AUTH:
  POST  /api/v1/auth/register/
  POST  /api/v1/auth/login/
  POST  /api/v1/auth/logout/
  POST  /api/v1/auth/token/refresh/         (reads HttpOnly refresh cookie)
  POST  /api/v1/auth/google/                (Google OAuth code exchange)
  POST  /api/v1/auth/verify-email/          { token }
  POST  /api/v1/auth/resend-verification/
  POST  /api/v1/auth/password/reset/        { email }
  POST  /api/v1/auth/password/reset/confirm/ { token, password }
  GET   /api/v1/auth/me/
  PATCH /api/v1/auth/me/

CHATBOT:
  GET   /api/v1/chatbot/                    Active chatbot info + customize settings
  POST  /api/v1/chatbot/upload/             Multipart — receives PDF, stores, calls FastAPI
  GET   /api/v1/chatbot/status/             { status, chunk_count, trained_at }
  POST  /api/v1/chatbot/chat/               SSE — proxies to FastAPI /chat/
  PATCH /api/v1/chatbot/customize/          { name, greeting, system_prompt, primary_color }
  GET   /api/v1/chatbot/history/            List all versions (is_active False ones)
  POST  /api/v1/chatbot/webhook/train-complete/  FastAPI calls this (X-Internal-Secret)

API KEY:
  GET   /api/v1/chatbot/api-key/            { prefix, masked_key, created_at, usage_count }
  POST  /api/v1/chatbot/api-key/rotate/     Deactivates old, creates new

CHAT SESSIONS (internal):
  GET   /api/v1/chatbot/sessions/           List user's test sessions
  POST  /api/v1/chatbot/sessions/           Create new session
  GET   /api/v1/chatbot/sessions/{id}/messages/

HEALTH:
  GET   /api/v1/health/                     { status: ok }

━━━ HOW DJANGO CALLS FASTAPI ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FastAPI runs at http://fastapi:8001 (Docker service name).
All Django→FastAPI calls use header: X-Internal-Secret: {INTERNAL_SECRET env var}

Calls Django makes to FastAPI:
  POST http://fastapi:8001/ingest/
    Body: { s3_key (or local_path in dev), chatbot_id, user_id }
    Fire-and-forget. Django returns 202 to frontend immediately.

  POST http://fastapi:8001/chat/
    Body: { chatbot_id, message, session_id, history: [...last 6 messages] }
    Django opens streaming httpx request, forwards SSE chunks to frontend.

  DELETE http://fastapi:8001/chatbot/{chatbot_id}
    Called when old chatbot version is soft-deleted (clean up vectors).

FastAPI calls Django:
  POST http://django:8000/api/v1/chatbot/webhook/train-complete/
    Body: { chatbot_id, status, chunk_count, total_pages, error? }
    Header: X-Internal-Secret

━━━ RESPONSE FORMAT (ALL ENDPOINTS) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Success:
  { "success": true, "data": {...}, "errors": null, "request_id": "uuid" }

Error:
  { "success": false, "data": null, "errors": [{"code": "ERR_001",
    "field": "email", "message": "human readable"}], "request_id": "uuid" }

━━━ KEY BUSINESS RULES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. One active chatbot per user. Enforced at DB level (partial unique index).
2. PDF must be validated: MIME type via magic bytes (not extension), max 10 MB.
3. In dev: store PDF to Docker volume. In prod: store to AWS S3.
   Use DJANGO_ENV env var to switch. Never hardcode S3 in local settings.
4. API key: store ONLY the SHA-256 hash. Full key generated once, returned
   once in creation response. Never retrievable again.
5. Version cleanup: Celery Beat task runs daily. Keeps 3 most recent versions
   per user. Soft-deletes older ones. Calls FastAPI DELETE to clean up vectors.
6. Email verification: user CANNOT access dashboard until verified.
   is_email_verified=False → 403 on all /chatbot/* endpoints.
7. Google OAuth users: is_email_verified automatically True (Google verified it).
8. Refresh token is set as HttpOnly, Secure, SameSite=Lax cookie by Django.
   Access token returned in response body only.
9. Internal Secret header must be validated on /webhook/train-complete/ endpoint.
   Reject with 403 if header missing or incorrect.

━━━ RULES — DO NOT SUGGEST ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✗ SQLite — we use PostgreSQL
✗ Django Channels / WebSocket — we use SSE
✗ Storing API key plaintext — SHA-256 hash only
✗ Synchronous training — fire-and-forget + webhook pattern
✗ Separate auth service — Django handles all auth
✗ Celery for chat streaming — chat is synchronous SSE proxy

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

## MASTER PROMPT 3 — RAG SERVICE (FastAPI)

```
MASTER CONTEXT — KORA AI RAG SERVICE (FASTAPI)

Project: Kora AI — SaaS RAG chatbot platform.
I need help with the FastAPI RAG microservice. Read all context before answering.
Do NOT give generic answers. Everything must fit this exact project.

━━━ WHAT THIS SERVICE DOES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FastAPI is the AI brain. It handles:
1. PDF ingestion: extract text → chunk → embed → store in pgvector
2. Internal chat: RAG retrieval + LLM streaming (called by Django)
3. Public embed chat: same RAG pipeline, called directly by the
   embedded JS widget on third-party websites (validated by API key)

FastAPI does NOT handle: user auth, chatbot metadata, billing, sessions.

━━━ TECH STACK ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Language:          Python 3.12
Framework:         FastAPI (latest stable)
PDF extraction:    PyMuPDF (fitz) — primary extractor
Text chunking:     LangChain RecursiveCharacterTextSplitter
                   chunk_size=500, chunk_overlap=50
                   separators=["\n\n", "\n", " ", ""]
Embeddings:        sentence-transformers — all-MiniLM-L6-v2 (local, free, 384 dims)
                   Loaded once at startup, runs inside FastAPI container
Vector DB:         pgvector in the same PostgreSQL instance Django uses
                   Raw asyncpg queries (NOT SQLAlchemy)
Primary LLM:       Groq API — llama-3.3-70b-versatile (free tier)
Fallback LLM:      Google Gemini 1.5 Flash (automatic if Groq fails/rate-limited)
LLM client:        groq Python SDK (primary), google-generativeai SDK (fallback)
Streaming:         FastAPI StreamingResponse (text/event-stream) for both LLMs
File access dev:   Read PDF from local Docker volume mount
File access prod:  Download PDF from AWS S3 via boto3
HTTP to Django:    httpx (async) for webhook callbacks
Rate limiting:     slowapi (Redis-backed) for /embed/chat/ endpoint
Config:            pydantic-settings (.env file)
Dev port:          8001

━━━ FILE STRUCTURE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

fastapi/
├── app/
│   ├── main.py               ← FastAPI app, CORS, router registration
│   ├── config.py             ← pydantic-settings (all env vars)
│   ├── database.py           ← asyncpg connection pool
│   ├── dependencies.py       ← Shared: verify internal secret, verify API key
│   ├── routers/
│   │   ├── ingest.py         ← POST /ingest/
│   │   ├── chat.py           ← POST /chat/ (internal)
│   │   ├── embed.py          ← POST /embed/chat/ (public)
│   │   └── management.py     ← DELETE /chatbot/{id}
│   ├── services/
│   │   ├── pdf_extractor.py  ← PyMuPDF text extraction
│   │   ├── chunker.py        ← LangChain text splitting
│   │   ├── embedder.py       ← sentence-transformers embedding
│   │   ├── vector_store.py   ← pgvector insert + similarity search
│   │   ├── llm_router.py     ← Groq primary, Gemini fallback logic
│   │   └── prompt_builder.py ← Builds final prompt with context + history
│   └── utils/
│       ├── storage.py        ← Local volume (dev) / S3 (prod) PDF access
│       └── webhook.py        ← httpx call back to Django
├── Dockerfile.dev
├── Dockerfile
└── requirements.txt

━━━ POSTGRESQL TABLE (FastAPI owns this) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TABLE: document_chunks
  id            UUID PK DEFAULT uuid_generate_v4()
  chatbot_id    UUID NOT NULL   ← no FK constraint (cross-service boundary)
  content       TEXT NOT NULL
  embedding     VECTOR(384) NOT NULL  ← 384 dims for MiniLM
  page_number   INTEGER NOT NULL DEFAULT 0
  chunk_index   INTEGER NOT NULL DEFAULT 0
  metadata      JSONB NOT NULL DEFAULT '{}'
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()

INDEXES:
  CREATE INDEX ON document_chunks (chatbot_id);
  CREATE INDEX ON document_chunks
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 50);

━━━ ALL FASTAPI ENDPOINTS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

INTERNAL (Header: X-Internal-Secret required):
  POST   /ingest/
    Body: { chatbot_id, local_path (dev) or s3_key (prod), user_id }
    Returns: 202 immediately. Runs ingestion as BackgroundTask.

  POST   /chat/
    Body: { chatbot_id, message, session_id, history: [{role, content}] }
    Returns: SSE stream (text/event-stream)

  DELETE /chatbot/{chatbot_id}
    Deletes all document_chunks for this chatbot_id.

PUBLIC (Header: X-API-Key required):
  POST   /embed/chat/
    Body: { chatbot_id, message, session_id }
    Returns: SSE stream
    Rate limited: 20 req/min per API key (Redis counter)
    CORS: Allow all origins (called from any third-party domain)

HEALTH:
  GET    /health/   → { status: "ok", model: "llama-3.3-70b", version: "1.0.0" }

━━━ INGESTION PIPELINE (step by step) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

POST /ingest/ returns 202 and runs as BackgroundTask:

1. Read PDF from local volume (dev) or download from S3 (prod)
2. Extract text using PyMuPDF page by page, track page numbers
3. Clean text: strip extra whitespace, fix common encoding issues
4. Split into chunks: RecursiveCharacterTextSplitter
   chunk_size=500, chunk_overlap=50
5. Delete existing chunks for this chatbot_id (handles re-training)
6. Generate embeddings for all chunks using sentence-transformers
   all-MiniLM-L6-v2 model (loaded at startup, runs locally)
   Process in batches of 32 to manage memory
7. Insert all (content, embedding, page_number, chunk_index) into
   document_chunks table via asyncpg executemany
8. Callback to Django: POST {DJANGO_URL}/api/v1/chatbot/webhook/train-complete/
   Body: { chatbot_id, status: "ready", chunk_count, total_pages }
   Header: X-Internal-Secret

On any exception: callback Django with { chatbot_id, status: "failed", error: "..." }

━━━ CHAT / RETRIEVAL PIPELINE (step by step) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Both /chat/ and /embed/chat/ use the same pipeline:

1. Embed the user's message using sentence-transformers (same model as ingestion)
2. pgvector similarity search:
   SELECT content, 1-(embedding <=> $query_vec) AS score
   FROM document_chunks WHERE chatbot_id=$id
   ORDER BY embedding <=> $query_vec LIMIT 5
3. Filter chunks: keep only those with score >= 0.3
4. Build prompt using prompt_builder.py:

   SYSTEM:
   "{user's custom system_prompt from DB}

   Answer questions based on the document context provided below.
   If the answer is not found in the context, use your general knowledge
   to help but clearly state: 'This information is not from your document.'
   Be concise and helpful. Bot name: {bot_name}."

   CONTEXT:
   "Relevant excerpts from the document:
   [chunk 1 content]
   ---
   [chunk 2 content] ..."

   HISTORY: last 6 messages from session

   USER: current message

5. Call LLM via llm_router.py:
   - Try Groq first (llama-3.3-70b-versatile, stream=True, max_tokens=600)
   - On Groq RateLimitError or APIError: automatically switch to
     Gemini 1.5 Flash (stream=True, max_output_tokens=600)
   - On both failing: return error SSE event

6. Stream SSE chunks: data: {"token": "...", "done": false}
   Final chunk: data: {"token": "", "done": true}

━━━ API KEY VALIDATION FOR /embed/chat/ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FastAPI validates the API key directly against PostgreSQL:
  1. Read X-API-Key header
  2. Extract prefix (first 8 chars after "kora_live_")
  3. Compute SHA-256 of full key
  4. Query: SELECT chatbot_id, is_active FROM api_keys
     WHERE key_prefix=$prefix AND key_hash=$hash AND is_active=TRUE
  5. If not found: return 401
  6. Verify chatbot_id in request body matches the key's chatbot_id
  7. Non-blocking: UPDATE api_keys SET last_used_at=NOW(),
     usage_count=usage_count+1 WHERE id=$id  (fire and forget)

━━━ RULES — DO NOT SUGGEST ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✗ Pinecone / Weaviate / Qdrant — we use pgvector in PostgreSQL
✗ OpenAI embeddings — we use sentence-transformers (free, local)
✗ OpenAI for LLM — we use Groq + Gemini (free tiers)
✗ Celery in FastAPI — use FastAPI BackgroundTasks
✗ SQLAlchemy — use raw asyncpg for vector operations
✗ LangChain chains/agents — use LangChain for text splitting only
✗ Synchronous ingestion endpoint — must return 202 immediately

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

## MASTER PROMPT 4 — DATABASE (PostgreSQL + pgvector)

```
MASTER CONTEXT — KORA AI DATABASE

Project: Kora AI — SaaS RAG chatbot platform.
I need help with the PostgreSQL database. Read all context before answering.
Do NOT give generic answers. Everything must fit this exact project.

━━━ DATABASE OVERVIEW ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

One PostgreSQL 16 instance shared by two services:
- Django (manages: users, chatbots, api_keys, sessions, messages)
- FastAPI (manages: document_chunks with pgvector embeddings)

No cross-service FK constraints at DB level. Referential integrity enforced
by application logic using UUID references.

━━━ REQUIRED EXTENSIONS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CREATE EXTENSION IF NOT EXISTS vector;          -- pgvector
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";     -- uuid_generate_v4()

These run on container startup via init.sql in Docker.

━━━ COMPLETE SCHEMA ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- DJANGO-MANAGED TABLES --

CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email             VARCHAR(254) UNIQUE NOT NULL,
  password_hash     VARCHAR(128) NOT NULL,
  full_name         VARCHAR(255) NOT NULL DEFAULT '',
  avatar            VARCHAR(500),
  is_active         BOOLEAN NOT NULL DEFAULT TRUE,
  is_staff          BOOLEAN NOT NULL DEFAULT FALSE,
  is_email_verified BOOLEAN NOT NULL DEFAULT FALSE,
  auth_provider     VARCHAR(10) NOT NULL DEFAULT 'email',
  google_id         VARCHAR(100),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at        TIMESTAMPTZ
);
CREATE INDEX ON users (email);
CREATE INDEX ON users (created_at);

CREATE TABLE chatbots (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name              VARCHAR(100) NOT NULL DEFAULT 'My Assistant',
  greeting_message  TEXT NOT NULL DEFAULT 'Hi! How can I help you today?',
  system_prompt     TEXT NOT NULL DEFAULT '',
  primary_color     VARCHAR(7) NOT NULL DEFAULT '#6366f1',
  is_active         BOOLEAN NOT NULL DEFAULT TRUE,
  status            VARCHAR(20) NOT NULL DEFAULT 'uploading'
                    CHECK (status IN ('uploading','processing','ready','failed')),
  pdf_filename      VARCHAR(255) NOT NULL,
  pdf_path          VARCHAR(500) NOT NULL,  -- local path (dev) or s3_key (prod)
  pdf_size_bytes    INTEGER NOT NULL,
  version_number    INTEGER NOT NULL DEFAULT 1,
  trained_at        TIMESTAMPTZ,
  total_pages       INTEGER,
  chunk_count       INTEGER,
  error_message     TEXT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at        TIMESTAMPTZ
);
-- Only 1 active chatbot per user at a time:
CREATE UNIQUE INDEX one_active_chatbot_per_user
  ON chatbots (user_id) WHERE deleted_at IS NULL AND is_active = TRUE;
CREATE INDEX ON chatbots (user_id, is_active);
CREATE INDEX ON chatbots (status);

CREATE TABLE api_keys (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  chatbot_id   UUID NOT NULL REFERENCES chatbots(id) ON DELETE CASCADE,
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  key_prefix   VARCHAR(8) NOT NULL,
  key_hash     VARCHAR(64) NOT NULL UNIQUE,
  is_active    BOOLEAN NOT NULL DEFAULT TRUE,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_used_at TIMESTAMPTZ,
  usage_count  BIGINT NOT NULL DEFAULT 0
);
CREATE UNIQUE INDEX one_key_per_chatbot
  ON api_keys (chatbot_id) WHERE is_active = TRUE;
CREATE INDEX ON api_keys (key_prefix);
CREATE INDEX ON api_keys (user_id, is_active);

CREATE TABLE chat_sessions (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  chatbot_id  UUID NOT NULL REFERENCES chatbots(id) ON DELETE CASCADE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON chat_sessions (user_id, chatbot_id);

CREATE TABLE chat_messages (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  session_id  UUID NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
  role        VARCHAR(10) NOT NULL CHECK (role IN ('user', 'assistant')),
  content     TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON chat_messages (session_id, created_at);

-- FASTAPI-MANAGED TABLE --

CREATE TABLE document_chunks (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  chatbot_id   UUID NOT NULL,       -- no FK, cross-service
  content      TEXT NOT NULL,
  embedding    VECTOR(384) NOT NULL, -- MiniLM all-MiniLM-L6-v2 = 384 dims
  page_number  INTEGER NOT NULL DEFAULT 0,
  chunk_index  INTEGER NOT NULL DEFAULT 0,
  metadata     JSONB NOT NULL DEFAULT '{}',
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON document_chunks (chatbot_id);
CREATE INDEX ON document_chunks
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 50);

━━━ KEY QUERIES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Vector similarity search (FastAPI):
  SELECT content, page_number,
         1 - (embedding <=> $1::vector) AS score
  FROM document_chunks
  WHERE chatbot_id = $2
  ORDER BY embedding <=> $1::vector
  LIMIT 5;
  -- Filter in application: keep only rows where score >= 0.3

Delete chatbot vectors (re-training or version cleanup):
  DELETE FROM document_chunks WHERE chatbot_id = $1;

API key lookup (FastAPI validates embed widget):
  SELECT id, chatbot_id, is_active
  FROM api_keys
  WHERE key_prefix = $1 AND key_hash = $2 AND is_active = TRUE;

Get user's active chatbot (Django):
  SELECT * FROM chatbots
  WHERE user_id = $1 AND is_active = TRUE AND deleted_at IS NULL
  LIMIT 1;

Get version history (Django):
  SELECT id, version_number, pdf_filename, status, trained_at, chunk_count
  FROM chatbots
  WHERE user_id = $1 AND is_active = FALSE AND deleted_at IS NULL
  ORDER BY version_number DESC
  LIMIT 3;

━━━ CONNECTIONS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Django:   psycopg2-binary via DATABASE_URL env var
          ATOMIC_REQUESTS=True (each request in a transaction)
FastAPI:  asyncpg connection pool (min=2, max=10)
          Both services use: postgresql://postgres:password@postgres:5432/koraai

━━━ RULES — DO NOT SUGGEST ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✗ A separate vector database — pgvector in PostgreSQL handles everything
✗ MySQL — we use PostgreSQL
✗ VECTOR(1536) — our model is MiniLM, use VECTOR(384)
✗ SQLAlchemy in FastAPI — use raw asyncpg
✗ WidthType.PERCENTAGE — use DXA for any doc generation
✗ Storing PDF content in the DB — store file path or S3 key only

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

## MASTER PROMPT 5 — DEVOPS (Docker + GitHub Actions)

```
MASTER CONTEXT — KORA AI DEVOPS

Project: Kora AI — SaaS RAG chatbot platform.
I need help with Docker, docker-compose, and GitHub CI/CD.
Read all context before answering. Do NOT give generic answers.

━━━ DEPLOYMENT CONTEXT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Development: Everything runs locally in Docker via docker-compose.
             This is the ONLY environment during MVP development phase.
Production:  AWS deployment — decided AFTER MVP is functionally complete.
             Do not build AWS-specific infra during MVP dev phase.

━━━ REPOSITORIES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Two GitHub repos:
  kora-ai-frontend   ← Next.js 14
  kora-ai-backend    ← Django + FastAPI (monorepo, two services)

kora-ai-backend/ structure:
  django/              ← Django project root
    manage.py
    config/
    apps/
    requirements/
      base.txt
      local.txt
    Dockerfile.dev     ← hot reload, mounts source
    Dockerfile         ← production (multi-stage)
  fastapi/             ← FastAPI service root
    app/
    requirements.txt
    Dockerfile.dev
    Dockerfile
  docker-compose.yml   ← full local stack
  .github/
    workflows/
      backend-ci.yml
  .env.example

━━━ DOCKER COMPOSE (FULL LOCAL STACK) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Services:
  postgres:
    image: pgvector/pgvector:pg16   ← official pgvector image (has extension built in)
    port: 5432
    env: POSTGRES_DB=koraai, POSTGRES_USER=postgres, POSTGRES_PASSWORD=postgres
    volume: postgres_data:/var/lib/postgresql/data
    init: runs ./docker/init.sql on first start
          (CREATE EXTENSION vector; CREATE EXTENSION "uuid-ossp";
           CREATE TABLE document_chunks ...)

  redis:
    image: redis:7-alpine
    port: 6379

  django:
    build: ./django (Dockerfile.dev)
    port: 8000
    volumes: ./django:/app (hot reload via runserver)
    env_file: ./django/.env
    depends_on: postgres, redis
    command: python manage.py runserver 0.0.0.0:8000

  fastapi:
    build: ./fastapi (Dockerfile.dev)
    port: 8001
    volumes: ./fastapi:/app (hot reload via uvicorn --reload)
    volumes: pdf_storage:/app/pdfs  ← shared PDF volume
    env_file: ./fastapi/.env
    depends_on: postgres, redis
    command: uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload

  celery:
    build: ./django (same Dockerfile.dev)
    volumes: ./django:/app
    volumes: pdf_storage:/app/pdfs  ← same shared volume
    env_file: ./django/.env
    depends_on: postgres, redis
    command: celery -A config worker -l INFO

  celery-beat:
    build: ./django (same Dockerfile.dev)
    env_file: ./django/.env
    depends_on: postgres, redis
    command: celery -A config beat -l INFO --scheduler django_celery_beat.schedulers:DatabaseScheduler

  nextjs:
    build: ../kora-ai-frontend (Dockerfile.dev) OR run separately
    port: 3000
    env_file: .env.local
    command: npm run dev

  pgadmin:
    image: dpage/pgadmin4
    port: 5050 (dev only, not in production compose)
    env: PGADMIN_DEFAULT_EMAIL=admin@koraai.com, PGADMIN_DEFAULT_PASSWORD=admin

Volumes: postgres_data, pdf_storage

━━━ SHARED PDF STORAGE IN DEV ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Both Django and FastAPI containers mount the same Docker volume: pdf_storage.
Django saves uploaded PDFs to /app/pdfs/{user_id}/{chatbot_id}.pdf
FastAPI reads from the same path /app/pdfs/{user_id}/{chatbot_id}.pdf
In production: Django uploads to S3, FastAPI downloads from S3 via boto3.
The switch is controlled by STORAGE_BACKEND env var: "local" | "s3"

━━━ ENV FILES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

django/.env (never committed, gitignored):
  DJANGO_SETTINGS_MODULE=config.settings.local
  SECRET_KEY=dev-secret-key
  DATABASE_URL=postgresql://postgres:postgres@postgres:5432/koraai
  REDIS_URL=redis://redis:6379/0
  FASTAPI_URL=http://fastapi:8001
  INTERNAL_SECRET=dev-internal-secret-change-in-prod
  DJANGO_URL=http://django:8000
  STORAGE_BACKEND=local
  PDF_STORAGE_PATH=/app/pdfs
  GOOGLE_CLIENT_ID=...
  GOOGLE_CLIENT_SECRET=...
  EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
  ALLOWED_HOSTS=localhost,127.0.0.1,django

fastapi/.env:
  DATABASE_URL=postgresql://postgres:postgres@postgres:5432/koraai
  REDIS_URL=redis://redis:6379/1
  INTERNAL_SECRET=dev-internal-secret-change-in-prod
  DJANGO_WEBHOOK_URL=http://django:8000
  STORAGE_BACKEND=local
  PDF_STORAGE_PATH=/app/pdfs
  GROQ_API_KEY=...
  GEMINI_API_KEY=...
  SENTENCE_TRANSFORMERS_HOME=/app/models  ← cache model inside container

.env.example committed for both (same keys, no values).

━━━ DOCKERFILE PATTERNS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Dockerfile.dev (Django):
  FROM python:3.12-slim
  WORKDIR /app
  COPY requirements/base.txt requirements/local.txt ./requirements/
  RUN pip install -r requirements/local.txt
  COPY . .
  # Source mounted as volume — no COPY needed for dev

Dockerfile.dev (FastAPI):
  FROM python:3.12-slim
  WORKDIR /app
  RUN apt-get update && apt-get install -y libgomp1  # sentence-transformers needs this
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  # Download model during build so it's cached in image:
  RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

━━━ GITHUB ACTIONS CI ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

backend-ci.yml — runs on PR to develop/main:

Django CI steps:
  1. Set up Python 3.12
  2. pip install -r django/requirements/local.txt
  3. flake8 django/apps django/config
  4. black --check django/
  5. isort --check-only django/
  6. pytest django/ --cov=apps --cov-fail-under=70

FastAPI CI steps:
  1. Set up Python 3.12
  2. pip install -r fastapi/requirements.txt
  3. ruff check fastapi/app/
  4. pytest fastapi/ --cov=app --cov-fail-under=65

frontend-ci.yml — runs on PR to develop/main (kora-ai-frontend repo):
  1. Node 20
  2. npm ci
  3. npx tsc --noEmit
  4. npx eslint src/
  5. npx next build

━━━ BRANCH STRATEGY ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

main         ← stable, deployable (protected, requires PR + CI)
develop      ← active development default branch
feature/*    ← feature work (feature/pdf-upload, feature/chat-ui)
fix/*        ← bug fixes

All 3 devs work on develop. Feature branches for larger units of work.
Minimum 1 PR review before merging to main.

━━━ COMMON COMMANDS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

docker-compose up                     # Start everything
docker-compose up --build             # Rebuild and start
docker-compose exec django python manage.py migrate
docker-compose exec django python manage.py createsuperuser
docker-compose exec django pytest
docker-compose exec fastapi pytest
docker-compose logs -f django         # Tail Django logs
docker-compose logs -f fastapi        # Tail FastAPI logs

━━━ RULES — DO NOT SUGGEST ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✗ Kubernetes / EKS — too complex for MVP
✗ AWS CDK / Terraform — deferred to post-MVP
✗ Jenkins — we use GitHub Actions
✗ Separate docker-compose per service — one file runs everything
✗ Hardcoding credentials in Dockerfiles — use .env files

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

## MASTER PROMPT 6 — AWS (Post-MVP, for future reference)

```
MASTER CONTEXT — KORA AI AWS INFRASTRUCTURE

Project: Kora AI — SaaS RAG chatbot platform.
NOTE: AWS deployment is DEFERRED — decided after MVP is complete in Docker.
This prompt is for planning and research questions about the future
AWS architecture. Do not build any AWS infra during the MVP phase.

━━━ PLANNED AWS ARCHITECTURE (POST-MVP) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Frontend:       AWS Amplify (Next.js 14 with SSR support)
Backend:        AWS ECS Fargate (Django + FastAPI as separate services)
Database:       AWS RDS PostgreSQL 16 with pgvector extension
Cache/Queue:    AWS ElastiCache Redis 7
File storage:   AWS S3 (private bucket for PDFs, public bucket via CloudFront
                for widget.js and Next.js static assets)
CDN:            AWS CloudFront (widget.js served globally)
Container repo: AWS ECR (one repo per service)
Email:          AWS SES (via django-anymail)
Secrets:        AWS Secrets Manager (all credentials)
DNS:            AWS Route 53
SSL:            AWS ACM (wildcard *.koraai.com)
Monitoring:     AWS CloudWatch

━━━ SERVICE URLS (PLANNED) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Frontend (Amplify):   https://koraai.com
Django API:           https://api.koraai.com
FastAPI RAG:          https://rag.koraai.com
Widget CDN:           https://cdn.koraai.com/widget.js
npm package:          @kora-ai/widget (published to npm registry)

━━━ S3 BUCKETS (PLANNED) ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

kora-ai-pdfs-{env}:   Private. Django uploads PDFs here.
                      FastAPI downloads via IAM role (no public access).
kora-ai-assets-{env}: CloudFront origin. Serves widget.js, static files.
                      Public read via CloudFront OAC only.

━━━ TRANSITION FROM DOCKER TO AWS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

The codebase is AWS-ready from day 1 via env vars:
  STORAGE_BACKEND=local → use Docker volume for PDFs
  STORAGE_BACKEND=s3    → use AWS S3 for PDFs
  EMAIL_BACKEND=console → print emails to terminal
  EMAIL_BACKEND=ses     → send via AWS SES

No code changes needed to deploy to AWS — only environment variable changes.
Production Docker images are multi-stage builds ready for ECS.

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

## MASTER PROMPT 7 — FULL STACK (Cross-Module Questions)

```
MASTER CONTEXT — KORA AI FULL PLATFORM

Project: Kora AI — SaaS RAG chatbot platform.
I need help with something that spans multiple services.
Read all context before answering. This is the complete picture.

━━━ PRODUCT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Name: Kora AI. Free SaaS platform. Users upload a PDF, train an AI chatbot,
test it in their dashboard, embed it on their own website via script tag or
React npm package. One active chatbot per user. Last 3 PDF versions archived.

━━━ SERVICES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Next.js 14 (port 3000) — frontend, App Router, TypeScript, Tailwind, shadcn/ui
  Zustand + TanStack Query. JWT in memory. Refresh token in HttpOnly cookie.
  Calls Django for everything. Never calls FastAPI directly.

Django 5 + DRF (port 8000) — backend, auth, business logic, orchestration
  JWT auth. Google OAuth. Email verification required. Celery tasks.
  Receives PDF, stores to local volume (dev) or S3 (prod), tells FastAPI.
  Proxies chat SSE from FastAPI to frontend.

FastAPI (port 8001) — RAG AI service
  PDF ingestion: PyMuPDF → LangChain chunker → sentence-transformers
  (all-MiniLM-L6-v2, 384 dims) → pgvector
  Chat: retrieve top-5 chunks (score ≥ 0.3) → build prompt → Groq primary
  (llama-3.3-70b) → Gemini 1.5 Flash fallback → SSE stream
  Public /embed/chat/ endpoint: called by widget on third-party sites.
  Validates API key against PostgreSQL api_keys table.

PostgreSQL 16 + pgvector (port 5432) — single database for both backends
  Django tables: users, chatbots, api_keys, chat_sessions, chat_messages
  FastAPI table: document_chunks (VECTOR(384) embeddings)

Redis 7 (port 6379) — Celery broker (db 0) + rate limiting (db 1)

All runs in Docker via docker-compose up. AWS deployment deferred post-MVP.

━━━ END-TO-END DATA FLOWS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PDF UPLOAD & TRAINING:
  1. User selects PDF in Next.js (client validates: PDF type, ≤10 MB)
  2. POST /api/v1/chatbot/upload/ → Django
  3. Django: validate magic bytes, store PDF to /app/pdfs/{user}/{chatbot}.pdf
  4. Django: create Chatbot record (status=processing), archive old version
  5. Django: POST http://fastapi:8001/ingest/ fire-and-forget (X-Internal-Secret)
  6. Django: return 202 to frontend with { chatbot_id, status: "processing" }
  7. Frontend: poll GET /api/v1/chatbot/status/ every 3 seconds
  8. FastAPI (background): read PDF → extract → chunk → embed → pgvector
  9. FastAPI: POST http://django:8000/api/v1/chatbot/webhook/train-complete/
  10. Django: update Chatbot status="ready", send "chatbot ready" email via Celery
  11. Frontend polling: detects "ready" → shows "Start Testing" button

INTERNAL CHAT TEST:
  1. User sends message in Next.js ChatWindow
  2. POST /api/v1/chatbot/chat/ → Django (SSE connection kept open)
  3. Django: validate user owns active chatbot
  4. Django: fetch last 6 messages from chat_sessions/messages
  5. Django: POST http://fastapi:8001/chat/ with {chatbot_id, message, history}
  6. FastAPI: embed query → pgvector top-5 → build prompt → Groq stream
  7. FastAPI: stream SSE chunks back to Django
  8. Django: forward each chunk to Next.js SSE response
  9. On stream end: Django saves user message + assistant response to DB

EMBED WIDGET ON THIRD-PARTY SITE:
  1. Third-party site loads widget.js from CDN
  2. Widget initializes with data-chatbot-id and data-api-key attributes
  3. Widget reads/creates session_id from localStorage
  4. Visitor sends message → widget POST http://rag.koraai.com/embed/chat/
     (direct browser → FastAPI, Django NOT involved)
  5. Header: X-API-Key: kora_live_...
  6. FastAPI: SHA-256 the key, query api_keys table, verify is_active
  7. FastAPI: rate check via Redis (20 req/min per key)
  8. FastAPI: retrieve chunks → build prompt → stream response
  9. Widget renders streaming tokens in chat bubble

━━━ SECRETS EACH SERVICE NEEDS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Both Django and FastAPI share: DATABASE_URL, INTERNAL_SECRET, STORAGE_BACKEND
Django adds: SECRET_KEY, REDIS_URL, FASTAPI_URL, GOOGLE_CLIENT_ID/SECRET,
             EMAIL_BACKEND, PDF_STORAGE_PATH, DJANGO_URL
FastAPI adds: GROQ_API_KEY, GEMINI_API_KEY, REDIS_URL, DJANGO_WEBHOOK_URL,
             PDF_STORAGE_PATH, SENTENCE_TRANSFORMERS_HOME
Next.js adds: NEXT_PUBLIC_API_URL, NEXT_PUBLIC_WIDGET_CDN_URL

━━━ MY QUESTION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[PASTE YOUR QUESTION HERE]
```

---

# SECTION 3 — QUICK REFERENCE

## Which Prompt to Use

| Question is about... | Use |
|---|---|
| Any Next.js page, component, hook, or UI behavior | Prompt 1 — Frontend |
| Any Django model, view, serializer, URL, or Celery task | Prompt 2 — Backend |
| PDF processing, embeddings, RAG retrieval, Groq/Gemini, pgvector | Prompt 3 — RAG Service |
| Database schema, SQL queries, pgvector setup, migrations | Prompt 4 — Database |
| Docker, docker-compose, Dockerfile, GitHub Actions CI | Prompt 5 — DevOps |
| Future AWS infrastructure planning | Prompt 6 — AWS |
| End-to-end flows, cross-service questions, architecture decisions | Prompt 7 — Full Stack |

---

## MVP Decisions at a Glance

| Decision | Confirmed Value |
|---|---|
| Product name | Kora AI |
| Billing in MVP | Free only |
| Login | Email+Password + Google OAuth |
| Email verification | Required before dashboard access |
| Chatbots per user | 1 active, last 3 versions archived |
| PDF limit | 10 MB |
| Re-upload | Archives old, creates new version |
| Version rollback | Not in MVP (read-only history) |
| PDF language | English only |
| Embeddings | sentence-transformers all-MiniLM-L6-v2 (free, local) |
| Primary LLM | Groq llama-3.3-70b-versatile |
| Fallback LLM | Google Gemini 1.5 Flash |
| Chatbot fallback behavior | LLM general knowledge + note it's not from PDF |
| Bot customization | Name + greeting + system prompt + primary color |
| Embed formats | Script tag + React npm package |
| Widget style | Floating bubble, bottom-right, user picks color |
| Embed session history | Server-side Redis, 24-hour TTL, session ID in localStorage |
| API key scope | One key per chatbot |
| Rate limit (embed) | 20 req/min per API key |
| Internal chat history | Saved per chatbot, reviewable |
| Marketing pages | All 6 (Home, Pricing, How It Works, Contact, Docs, Blog) |
| Admin panel | Django admin |
| Target users | Both technical and non-technical |
| Transactional emails | Welcome, verify, password reset, chatbot ready, training failed |
| Timeline | 4–6 weeks |
| Dev environment | Docker (docker-compose) |
| AWS deployment | Deferred post-MVP |

---

## Change Log

| Date | Change | Decided By |
|---|---|---|
| March 2026 | Initial MVP spec locked | Founding team |

*Add all future spec changes to this table with a date and reason.*

---

*Kora AI MVP Specification — Version 1.0 — Locked March 2026*
*Do not modify without team agreement. Log all changes above.*
