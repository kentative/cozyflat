# Proposed System Design

## 1. Architecture Overview

CozyFlat is a multi-tenant property management platform built around four autonomous AI agents. The system follows a **modular monorepo** structure with a clear separation between the client layer, an API gateway, an agent orchestration layer, and shared backend services.

```
┌─────────────────────────────────────────────────┐
│               Client Layer                       │
│   Next.js (Web)      React Native / Expo (Mobile)│
└──────────────────────┬──────────────────────────┘
                       │ HTTPS / WebSocket
┌──────────────────────▼──────────────────────────┐
│              API Gateway                         │
│         Next.js API Routes + tRPC               │
│  Auth │ Rate Limiting │ Request Routing          │
└──────────────────────┬──────────────────────────┘
                       │ REST / gRPC
┌──────────────────────▼──────────────────────────┐
│          Agent Orchestration Layer               │
│              FastAPI + LangGraph                 │
│  ┌───────────┐ ┌──────────┐ ┌────────────────┐  │
│  │Maintenance│ │Financial │ │ Compliance &   │  │
│  │  Agent    │ │  Agent   │ │  Legal Agent   │  │
│  └───────────┘ └──────────┘ └────────────────┘  │
│              ┌──────────────────┐                │
│              │ Communication Hub│                │
│              └──────────────────┘                │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│               Shared Services Layer              │
│  Supabase │ Pinecone │ Redis │ Supabase Storage  │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│            External Integrations                 │
│  Stripe │ Twilio │ Checkr │ TaskRabbit │ Thumbtack│
│  DocuSign │ Smart Lock APIs │ IRS / Tax APIs     │
└─────────────────────────────────────────────────┘
```

---

## 2. Agent Designs

### 2.1 AI Maintenance Agent

**Responsibility:** End-to-end maintenance lifecycle from intake to vendor closeout.

**Flow:**
```
Tenant submits request (photo/video + text)
        ↓
Media uploaded to Supabase Storage
        ↓
GPT-4o Vision analyzes media → issue classification + severity score
        ↓
LangGraph routes to dispatch sub-agent
        ↓
TaskRabbit / Thumbtack API → vendor shortlist (by proximity, rating, cost)
        ↓
Landlord approval (push notification) → confirmed booking
        ↓
Smart lock API issues time-bound access code to vendor
        ↓
Vendor completes job → photo upload → GPT-4o verifies completion
        ↓
Auto-close ticket + log expense to Financial Agent
```

**Key Components:**
- `MaintenanceAgent` (LangGraph StateGraph) — manages state across intake → dispatch → closeout
- Media pipeline: Supabase Storage → GPT-4o Vision → structured `IssueReport` object
- Vendor selection tool: wraps TaskRabbit/Thumbtack APIs with cost/availability scoring
- Smart lock tool: wraps August/Yale/Schlage API for temporary PIN generation

**Data Models:**
```
MaintenanceRequest { id, property_id, tenant_id, severity, category,
                     media_urls[], status, vendor_id, access_code, created_at }
Vendor             { id, platform, name, rating, hourly_rate, skills[], location }
```

---

### 2.2 AI Financial Agent

**Responsibility:** Automated bookkeeping, tax prep, rent collection, and payouts.

**Flow:**
```
Income / expense events (rent payments, vendor invoices, maintenance costs)
        ↓
Financial Agent classifies each transaction → Schedule E category mapping
        ↓
Ledger entries written to PostgreSQL (double-entry bookkeeping schema)
        ↓
ML delinquency model scores each tenant monthly
        ↓
Automated reminder sequences via Twilio (SMS/voice) at D-3, D-0, D+5
        ↓
Stripe Connect disbursements → property owner net proceeds
```

**Key Components:**
- `FinancialAgent` (LangGraph) — orchestrates ledgering, collections, payouts
- Schedule E mapper: LLM-assisted categorization against IRS Publication 527 rules stored in Pinecone
- Delinquency predictor: lightweight ML model (scikit-learn / XGBoost) trained on payment history features
- Payout engine: Stripe Connect Transfer API with hold logic for escrow/security deposits

**Data Models:**
```
Transaction    { id, property_id, type, amount, schedule_e_category,
                 stripe_transfer_id, date, memo }
LedgerEntry    { id, transaction_id, account, debit, credit }
DelinquencyScore { tenant_id, score, risk_tier, evaluated_at }
```

---

### 2.3 Compliance & Legal Agent

**Responsibility:** Jurisdiction-aware lease creation and automated legal document delivery.

**Flow:**
```
New tenancy initiated (property location + tenant details)
        ↓
Compliance Agent queries Pinecone for applicable state/local laws
        ↓
GPT-4o / Claude 3.5 populates lease template with jurisdiction-specific clauses
        ↓
Landlord reviews → approves in app
        ↓
DocuSign API sends to tenant for e-signature
        ↓
Signed document stored in Supabase Storage + metadata indexed
        ↓
Agent tracks renewal / notice deadlines → auto-serves notices via DocuSign + Twilio
```

**Key Components:**
- `ComplianceAgent` (LangGraph) — lease gen, notice scheduling, deadline tracking
- Law RAG pipeline: jurisdiction laws + landlord-tenant statutes ingested into Pinecone
- Lease template engine: structured prompt with variable injection (state, city, rent amount, terms)
- Notice scheduler: cron-driven service checks upcoming notice deadlines and triggers delivery

**Data Models:**
```
Lease       { id, property_id, tenant_id, jurisdiction, status, docusign_envelope_id,
              effective_date, expiry_date, clauses_json }
LegalNotice { id, lease_id, type, served_at, delivery_method, acknowledged_at }
```

---

### 2.4 Communication Hub

**Responsibility:** Structured, AI-moderated messaging between landlords and tenants with automatic emergency escalation.

**Flow:**
```
Message sent by landlord or tenant
        ↓
AI moderation layer (Claude 3.5 Haiku for speed) checks tone + content policy
        ↓
If flagged → message held, sender notified to revise
        ↓
If emergency keywords / sentiment detected → escalation sub-agent fires
        ↓
Emergency escalation: push notification → Twilio voice call → maintenance ticket
        ↓
Normal message → delivered via Supabase Realtime websocket
```

**Key Components:**
- `CommunicationAgent` (LangGraph) — moderation + escalation routing
- Moderation tool: Claude 3.5 Haiku inline call (fast, cheap) for every message
- Emergency classifier: intent detection model / prompt chain — distinguishes fire/flood/medical from urgent maintenance
- Realtime delivery: Supabase Realtime channels (one channel per property unit)

**Data Models:**
```
Message     { id, thread_id, sender_id, content, moderation_status,
              is_emergency, created_at }
Thread      { id, property_id, participants[], type }
Escalation  { id, message_id, channel, resolved_at }
```

---

## 3. Tech Stack (Final Proposed)

### Frontend
| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Web app | Next.js 14 (App Router) | SSR for SEO on marketing pages; RSC for dashboard |
| Mobile | React Native + Expo | Code sharing with web hooks/utils; OTA updates |
| State management | Zustand + React Query | Lightweight global state; server-state caching |
| UI components | shadcn/ui (web), Tamagui (mobile) | Consistent design system across platforms |
| Type safety | TypeScript + tRPC | End-to-end type safety from API to client |

### API Gateway
| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Gateway | Next.js API Routes + tRPC | Collocated with web app; zero-config RPC |
| Auth | Supabase Auth (JWT + RLS) | Row-level security enforced at DB layer |
| Rate limiting | Upstash Redis | Serverless-compatible rate limiter |

### Agent Orchestration
| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Framework | FastAPI (Python 3.12) | Async, high-performance; native LangChain ecosystem |
| Orchestration | LangGraph | Stateful agent graphs; built-in checkpointing and human-in-the-loop |
| Primary model | GPT-4o | Vision support (maintenance triage), strong reasoning |
| Secondary model | Claude 3.5 Sonnet | Lease generation, document analysis, long-context |
| Moderation model | Claude 3.5 Haiku | High-speed inline message moderation |
| Embeddings | text-embedding-3-large | Laws, manuals, and lease clauses in Pinecone |

### Backend / Data
| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Primary DB | Supabase (PostgreSQL 15) | Auth, RLS, Realtime, and Storage in one platform |
| Vector DB | Pinecone (serverless) | Semantic search over laws, manuals, and lease clauses |
| Cache / Queue | Upstash Redis | Job queues (BullMQ), caching, pub/sub |
| File storage | Supabase Storage | Maintenance photos, signed leases, documents |
| Background jobs | BullMQ (Node) + Celery (Python) | Scheduled tasks (notices, reminders, payouts) |

### Infrastructure & DevOps
| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Web hosting | Vercel | Zero-config Next.js deployments; edge network |
| Agent API | Railway or Fly.io | Persistent Python services with GPU-optional workers |
| CI/CD | GitHub Actions | Monorepo-aware pipelines; deploy on merge |
| Secrets | Doppler | Centralized secrets across all environments |
| Observability | Langfuse (LLM traces) + Sentry | Agent trace visibility; error tracking |
| Monorepo | Turborepo | Shared packages, build caching across web/mobile/api |

### External Integrations
| Purpose | Service | Notes |
|---------|---------|-------|
| P2P Payments | Stripe Connect | Custom accounts for owners; instant payouts |
| SMS / Voice | Twilio | Rent reminders, emergency calls, notice delivery |
| Background checks | Checkr | Tenant screening on application |
| Vendor dispatch | TaskRabbit API + Thumbtack API | Parallel quote requests; auto-book on approval |
| E-signature | DocuSign | Lease signing, notice acknowledgment |
| Smart locks | August / Yale Connect API | Time-bound vendor access codes |

---

## 4. Data Flow & Key Integrations

### Authentication & Multi-tenancy
- Supabase Auth manages users with roles: `landlord`, `tenant`, `vendor`, `admin`
- PostgreSQL Row-Level Security (RLS) enforces property-scoped data isolation
- Every agent call carries a JWT; the agent layer validates scope before acting

### Agent-to-Agent Communication
- Agents communicate via **shared Supabase tables** (event sourcing pattern)
- Maintenance Agent publishes expense events → Financial Agent consumes them
- Compliance Agent publishes lease events → Communication Hub creates the initial thread
- LangGraph checkpointer uses Redis for in-flight agent state

### File Handling
- All media (photos, videos, documents) stored in Supabase Storage with signed URLs
- Virus scanning via ClamAV sidecar before files are processed by AI agents
- Retention policy: maintenance media 2 years, legal documents 7 years (IRS requirement)

---

## 5. Directory Structure (Monorepo)

```
cozyflat/
├── apps/
│   ├── web/                  # Next.js web app
│   └── mobile/               # React Native / Expo app
├── packages/
│   ├── ui/                   # Shared UI components
│   ├── db/                   # Supabase schema, migrations, typed client
│   └── shared/               # Shared types, utils, constants
├── services/
│   ├── agent-api/            # FastAPI + LangGraph agent service
│   │   ├── agents/
│   │   │   ├── maintenance/
│   │   │   ├── financial/
│   │   │   ├── compliance/
│   │   │   └── communication/
│   │   ├── tools/            # Agent tools (Stripe, Twilio, etc.)
│   │   └── rag/              # Pinecone ingestion + retrieval
│   └── jobs/                 # Background job workers (BullMQ/Celery)
├── docs/
│   ├── intent.md
│   ├── core-requirements.md
│   └── proposed-design.md
└── turbo.json
```

---

## 6. Key Design Decisions & Trade-offs

| Decision | Choice | Alternative Considered | Rationale |
|----------|--------|----------------------|-----------|
| Agent framework | LangGraph | CrewAI, AutoGen | Best-in-class stateful graphs; human-in-the-loop built in; active development |
| Database | Supabase | Firebase, PlanetScale | Auth + RLS + Realtime + Storage in one; avoids vendor sprawl |
| API style | tRPC | REST, GraphQL | End-to-end type safety with zero schema drift; simpler than GraphQL for this use case |
| Mobile | React Native + Expo | Flutter, PWA | Code/logic sharing with Next.js; large ecosystem; EAS build pipeline |
| E-signature | DocuSign | HelloSign, PandaDoc | Enterprise compliance; legally admissible in all 50 states |
| Lease RAG | Pinecone | pgvector | Purpose-built vector DB scales better for multi-jurisdiction law ingestion |
| LLM routing | GPT-4o + Claude 3.5 | Single-model | GPT-4o for vision tasks; Claude 3.5 for long-document analysis; Haiku for moderation speed/cost |
