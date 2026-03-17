# Revised System Design

## Problems with the Previous Design

Before the proposal, it's worth being explicit about what the previous design gets wrong:

| Issue | Previous Design | Impact |
|-------|----------------|--------|
| **Polyglot split** | FastAPI (Python) + Next.js (Node) | Two runtimes, two deployment targets, cross-language type drift, two job queue systems (BullMQ + Celery) |
| **tRPC overhead** | tRPC on top of Next.js | Redundant in Next.js 15 — Server Actions + Zod give the same end-to-end type safety natively |
| **Unnecessary vendor** | Pinecone as separate vector DB | pgvector ships inside Supabase; the law/manual corpus doesn't justify a dedicated vector DB at this scale |
| **Expensive e-signature** | DocuSign | $50–400/month before volume; legally identical open-source alternatives exist |
| **Dual job queues** | BullMQ (Node) + Celery (Python) | Two systems to operate, monitor, and debug — a maintenance risk from day one |
| **Multi-model routing complexity** | GPT-4o + Claude 3.5 Sonnet + Claude 3.5 Haiku | Three API clients, different token limits, different pricing, routing logic to maintain |
| **Separate ML model** | scikit-learn/XGBoost for delinquency | Requires labeled training data, model retraining pipeline, and a serving endpoint — premature for an early-stage product |
| **Secrets sprawl** | Doppler | Platform-level env vars (Vercel, Supabase) are sufficient; Doppler is operational overhead, not a feature |

---

## 1. Architecture Overview

**Philosophy: TypeScript-native, serverless-first, minimal vendor footprint.**

The revised design eliminates the Python service entirely. All agent logic runs as TypeScript in Vercel serverless functions, co-located with the application. Background jobs are managed by Trigger.dev v3 — a single durable job system that replaces both BullMQ and Celery. The vector store moves into Supabase via pgvector, removing Pinecone.

```
┌──────────────────────────────────────────────────────┐
│                     Client Layer                      │
│   Next.js 15 (Web)          Expo + Expo Router (Mobile)│
│   shadcn/ui                 NativeWind v4             │
└───────────────────────────┬──────────────────────────┘
                            │ Server Actions / REST
┌───────────────────────────▼──────────────────────────┐
│              Next.js 15 App Layer (Vercel)            │
│   Server Actions │ API Routes │ Edge Middleware       │
│   Supabase Auth (JWT + RLS) │ Upstash rate limiting  │
└───────────────────────────┬──────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────┐
│       Agent Layer  —  Mastra + Vercel AI SDK          │
│                                                       │
│  ┌──────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ Maintenance  │  │  Financial  │  │ Compliance  │  │
│  │   Agent      │  │   Agent     │  │ & Legal     │  │
│  └──────────────┘  └─────────────┘  └─────────────┘  │
│           ┌──────────────────────────┐                │
│           │     Communication Hub    │                │
│           └──────────────────────────┘                │
│                                                       │
│   Claude 3.7 Sonnet  (primary — vision, reasoning,   │
│                        long documents, tool use)      │
│   Claude 3.5 Haiku   (moderation — latency-critical) │
└───────────────────────────┬──────────────────────────┘
                            │  enqueue
┌───────────────────────────▼──────────────────────────┐
│           Trigger.dev v3  (Durable Job Runtime)       │
│   Payout tasks │ Notice scheduler │ Reminder sequences│
│   Delinquency scoring │ Vendor follow-up              │
└───────────────────────────┬──────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────┐
│                      Data Layer                       │
│   Supabase PostgreSQL + pgvector (laws, manuals)      │
│   Supabase RLS + Realtime + Storage                   │
│   Upstash Redis  (rate limiting, ephemeral cache)     │
└───────────────────────────┬──────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────┐
│                 External Integrations                 │
│   Stripe Connect │ Twilio │ Checkr                   │
│   TaskRabbit + Thumbtack │ Smart Lock Adapter        │
│   Docuseal  (e-signature, open source)               │
└──────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

### Frontend
| Layer | Technology | Why |
|-------|-----------|-----|
| Web | Next.js 15 (App Router, PPR, Server Actions) | Server Actions replace tRPC; Partial Prerendering for dashboard performance |
| Mobile | Expo + Expo Router v3 | File-based routing mirrors Next.js; OTA updates via EAS; code-sharing with web |
| Web UI | shadcn/ui + Tailwind CSS v4 | Unstyled primitives; full design control |
| Mobile UI | NativeWind v4 | Tailwind in React Native; consistent token system with web |
| Client state | Zustand + TanStack Query v5 | Lightweight global state; server-state caching with stale-while-revalidate |
| Forms | React Hook Form + Zod | Type-safe forms with schema validation shared between client and server |

**Note on tRPC removal:** Next.js 15 Server Actions provide the same end-to-end type safety as tRPC with zero additional dependencies. The pattern `const result = await serverAction(input)` infers types automatically. tRPC is only justified when you need a public API consumed by third parties — that's not the case here.

### AI Layer
| Layer | Technology | Why |
|-------|-----------|-----|
| SDK | Vercel AI SDK 4.x | Streaming, tool calls, `generateObject` for structured output, multi-step agents — native to Next.js |
| Agent framework | Mastra | TypeScript-native agent framework with workflow engine, persistent memory, and built-in RAG; runs in serverless functions — no separate service |
| Primary model | Claude 3.7 Sonnet | Vision (maintenance photos), long-context (lease documents, 200k tokens), structured output, tool use — one model replaces GPT-4o + Claude 3.5 |
| Moderation model | Claude 3.5 Haiku | Still justified for the Communication Hub — latency-critical, sub-200ms, cheap at scale |
| Embeddings | `text-embedding-3-small` (OpenAI) | Best price/performance for the law/manual corpus in pgvector |
| Observability | Langfuse | LLM trace logging, cost tracking per agent |

**Note on model consolidation:** Claude 3.7 Sonnet handles vision natively (maintenance triage), has 200k context (full lease documents), and produces reliable structured JSON output. The original design's three-model routing (GPT-4o vision → Claude Sonnet documents → Claude Haiku moderation) is unnecessary — it adds API client management and routing logic for marginal capability gain.

### Backend & Data
| Layer | Technology | Why |
|-------|-----------|-----|
| Primary DB | Supabase (PostgreSQL 16) | Auth + RLS + Realtime + Storage + pgvector in one platform |
| Vector search | pgvector (Supabase extension) | Law corpus + property manuals fit comfortably in pgvector; eliminates Pinecone as a separate vendor and billing surface |
| Job runtime | Trigger.dev v3 | Single durable job system replacing BullMQ + Celery; serverless-native; full observability dashboard; retries and dead-letter queues built in |
| Cache | Upstash Redis | Serverless-compatible; rate limiting on API routes; ephemeral session caching |
| File storage | Supabase Storage | Maintenance media, signed leases, legal notices — with signed URLs and per-bucket retention policies |

**Note on pgvector vs. Pinecone:** The law corpus is a fixed-size, mostly-static dataset (50 state statutes + key cities). pgvector with `ivfflat` or `hnsw` indexing handles millions of vectors with sub-50ms recall at this scale. Pinecone's advantage (massive scale, multi-tenant isolation) is irrelevant here and costs $70–700/month extra.

### Infrastructure
| Layer | Technology | Why |
|-------|-----------|-----|
| Hosting | Vercel | Next.js deployments; edge middleware for auth; serverless functions for agents |
| Background jobs | Trigger.dev v3 (cloud) | Collocated job dashboard; no separate infra to manage |
| CI/CD | GitHub Actions | Turborepo-aware; deploy-on-merge with preview URLs |
| Observability | Langfuse + Sentry | LLM traces + application errors |
| Monorepo | Turborepo + pnpm workspaces | Shared packages, remote build caching |

**Note on deployment simplicity:** The previous design required two hosting platforms (Vercel for Next.js, Railway/Fly.io for FastAPI). This design runs entirely on Vercel, with Trigger.dev as the only additional cloud service. No Python container to size, scale, or monitor.

### External Integrations
| Purpose | Service | Notes |
|---------|---------|-------|
| Payments | Stripe Connect | Custom connected accounts for owners; instant payouts; webhook events drive Financial Agent |
| SMS / Voice | Twilio | Rent reminders, emergency voice calls, notice delivery |
| Background checks | Checkr | Tenant screening; webhook on completion |
| Vendor dispatch | TaskRabbit + Thumbtack APIs | Parallel quote requests via Maintenance Agent tools |
| Smart locks | `SmartLockAdapter` (abstraction) | Unified interface over August, Yale, Schlage, Latch — swap lock brand without touching agent code |
| E-signature | Docuseal | Open-source DocuSign alternative; legally valid in all 50 states; self-hosted option for zero per-envelope cost |

**Note on Docuseal vs. DocuSign:** DocuSign charges per envelope ($1–2+) and has a complex enterprise contract. Docuseal is MIT-licensed, API-compatible, and can be self-hosted on a $6/month VPS or used via their cloud at a fraction of the cost. For a product that generates leases and notices at scale, this matters.

---

## 3. Agent Designs

### 3.1 AI Maintenance Agent

**Orchestration:** Mastra workflow (durable, resumable state machine)

```
Tenant submits ticket (text + photos/video)
        ↓
Supabase Storage upload → signed URLs returned
        ↓
Claude 3.7 Sonnet (vision) → IssueReport { category, severity, urgency, description }
        ↓
[severity === 'emergency'] → skip approval, auto-dispatch + notify landlord
[severity !== 'emergency'] → notify landlord for approval
        ↓
Mastra workflow pauses (awaits landlord approval event)  ← human-in-the-loop
        ↓
On approval: TaskRabbit + Thumbtack tools run in parallel → vendor shortlist
        ↓
Landlord selects vendor (or agent auto-selects if landlord non-responsive > 4h)
        ↓
SmartLockAdapter.createTemporaryPin(vendorId, window) → PIN issued to vendor via SMS
        ↓
Trigger.dev schedules vendor follow-up check at job completion window
        ↓
Vendor uploads completion photos → Claude 3.7 Sonnet verifies work vs. original issue
        ↓
Ticket closed → expense event emitted → Financial Agent ledgers the cost
        ↓
SmartLockAdapter.revokePin() → access revoked
```

**Key design changes from original:**
- Mastra workflow is resumable across process restarts — if the server restarts while waiting for landlord approval, the workflow resumes from the checkpoint
- Emergency bypass lane removes friction in fire/flood scenarios
- Auto-select fallback prevents tickets stalling when landlords are unresponsive
- `SmartLockAdapter` abstracts lock brand so the agent is brand-agnostic

### 3.2 AI Financial Agent

**Orchestration:** Trigger.dev scheduled jobs + Vercel AI SDK for classification

```
Stripe webhook (payment/payout/refund) → Trigger.dev event
        ↓
Claude 3.7 Sonnet classifies transaction → Schedule E category
  (IRS Pub 527 rules embedded in system prompt, no RAG needed for this)
        ↓
Double-entry ledger rows written to PostgreSQL
        ↓
[Monthly cron] Trigger.dev job: pull 12-month payment history per tenant
        ↓
Claude 3.7 Sonnet: analyze patterns → DelinquencyAssessment { score, tier, reasoning, action }
        ↓
[score > threshold] → enqueue Twilio reminder sequence (D-3, D0, D+5, D+10)
        ↓
[rent_due event] → Stripe Connect Transfer → owner net proceeds disbursed
```

**Key design change — delinquency prediction:**
The original proposed scikit-learn/XGBoost, which requires labeled training data (you don't have at launch), a feature engineering pipeline, model retraining jobs, and a serving endpoint. Instead:

Claude 3.7 Sonnet receives structured payment history (12 months of payment dates vs. due dates, partial payments, NSFs, communication responsiveness) and returns a structured risk assessment with a score, tier, and **reasoning**. The reasoning is more actionable than a raw probability score — it tells the landlord *why* a tenant is flagged, not just *that* they are. Migrate to a proper ML model when you have thousands of tenants and labeled outcomes.

### 3.3 Compliance & Legal Agent

**Orchestration:** Mastra workflow + pgvector RAG

```
New tenancy event (property_id, tenant_id, jurisdiction)
        ↓
pgvector similarity search → retrieve applicable state + city statutes
  (landlord-tenant law corpus pre-ingested with text-embedding-3-small)
        ↓
Claude 3.7 Sonnet (200k context): populate lease template with jurisdiction clauses
  → structured LeaseDocument { sections[], clauses[], disclosures[] }
        ↓
Landlord reviews in app → requests edits or approves
        ↓
Mastra workflow pauses (awaits landlord approval)  ← human-in-the-loop
        ↓
Docuseal API: create envelope → tenant receives signing request
        ↓
Docuseal webhook: signed → PDF stored in Supabase Storage
        ↓
Trigger.dev schedules: renewal notice (60 days before expiry), move-out notice deadlines
        ↓
[deadline reached] → Docuseal sends notice + Twilio SMS delivery confirmation
```

**Key design change — pgvector over Pinecone:**
The law corpus is ingested once per jurisdiction and updated quarterly. pgvector with HNSW indexing on a Supabase instance handles this comfortably. The retrieval query is: "given this city + state, find all applicable landlord-tenant statutes." That's a well-bounded semantic search, not a high-throughput multi-tenant vector workload.

### 3.4 Communication Hub

**Orchestration:** Supabase Realtime + inline Claude 3.5 Haiku moderation

```
User sends message (landlord or tenant)
        ↓
Next.js Server Action receives message
        ↓
Claude 3.5 Haiku: moderate({ content, history })
  → { decision: 'allow' | 'hold' | 'escalate', reason?, emergency_type? }
        ↓
[decision === 'hold']     → return to sender with revision request
[decision === 'escalate'] → emergency pipeline:
    • Supabase Realtime push to all participants
    • Trigger.dev job: Twilio voice call to landlord
    • Auto-create Maintenance ticket at severity='emergency'
    • Log escalation event
[decision === 'allow']    → write to messages table
    • Supabase Realtime delivers to recipient via WebSocket
        ↓
[unread > 2h] → Trigger.dev job enqueues Twilio SMS nudge
```

**Key design change:** The original used LangGraph for moderation routing. Haiku moderation is a single fast inference call — it doesn't need a state graph. Moving this into a Next.js Server Action means zero cold start, zero separate service, and the result is available synchronously before the message is committed to the DB.

---

## 4. Data Models

```sql
-- Core multi-tenancy
organizations   { id, name, plan, stripe_customer_id }
properties      { id, org_id, address, jurisdiction, smart_lock_config jsonb }
units           { id, property_id, number, floor }
users           { id, org_id, role: landlord|tenant|vendor|admin, ... }
tenancies       { id, unit_id, tenant_id, start_date, end_date }

-- Maintenance
maintenance_requests {
  id, unit_id, tenant_id, status, severity, category,
  media_urls text[], vendor_id, access_pin_hash,
  workflow_run_id,  -- Mastra workflow reference
  expense_id, created_at, closed_at
}

-- Financial
transactions {
  id, property_id, type, amount_cents, schedule_e_category,
  stripe_event_id, date, memo, created_by
}
ledger_entries   { id, transaction_id, account, debit_cents, credit_cents }
delinquency_scores { tenant_id, score, tier, reasoning text, evaluated_at }

-- Legal
leases {
  id, unit_id, tenant_id, jurisdiction, status,
  docuseal_envelope_id, document_url,
  effective_date, expiry_date, clauses jsonb
}
legal_notices {
  id, lease_id, type, docuseal_envelope_id,
  scheduled_at, served_at, acknowledged_at, delivery_method
}

-- Communication
threads   { id, unit_id, type: maintenance|general|legal, participants uuid[] }
messages  {
  id, thread_id, sender_id, content_encrypted,
  moderation_status, is_emergency, emergency_type,
  created_at, delivered_at
}
escalations { id, message_id, type, resolved_at, resolution_notes }

-- Vector store (pgvector)
law_chunks {
  id, jurisdiction, statute_ref, content text,
  embedding vector(1536), updated_at
}
```

---

## 5. Directory Structure

```
cozyflat/
├── apps/
│   ├── web/                        # Next.js 15
│   │   ├── app/
│   │   │   ├── (dashboard)/        # Landlord dashboard
│   │   │   ├── (tenant)/           # Tenant portal
│   │   │   ├── api/                # Webhooks (Stripe, Twilio, Docuseal, Checkr)
│   │   │   └── actions/            # Server Actions (replace tRPC)
│   │   └── agents/                 # Mastra agent definitions
│   │       ├── maintenance.ts
│   │       ├── financial.ts
│   │       ├── compliance.ts
│   │       └── communication.ts
│   └── mobile/                     # Expo + Expo Router v3
│       └── app/                    # File-based routes
├── packages/
│   ├── ui/                         # shadcn/ui components (web)
│   ├── db/                         # Supabase schema, migrations, typed client
│   ├── ai/                         # Shared Vercel AI SDK configs, prompts, schemas
│   └── shared/                     # Shared Zod schemas, types, constants
├── trigger/                        # Trigger.dev v3 jobs
│   ├── financial/
│   │   ├── delinquency-scorer.ts
│   │   ├── payout-processor.ts
│   │   └── reminder-sequence.ts
│   ├── legal/
│   │   └── notice-scheduler.ts
│   └── maintenance/
│       └── vendor-followup.ts
├── docs/
│   ├── core-requirements.md
│   ├── proposed-design.md          # Original (kept for reference)
│   └── revised-design.md           # This document
└── turbo.json
```

---

## 6. What Was Removed and Why

| Removed | Replaced With | Saving |
|---------|--------------|--------|
| FastAPI (Python service) | Mastra agents in Next.js serverless functions | Eliminates Python runtime, Railway/Fly.io hosting, cross-language types |
| LangGraph | Mastra | TypeScript-native; no Python dependency; same stateful workflow capability |
| tRPC | Next.js 15 Server Actions + Zod | Zero extra dependencies; same type safety; native to the framework |
| Pinecone | pgvector (Supabase extension) | $70–700/month saved; one less vendor; law corpus fits comfortably |
| BullMQ + Celery | Trigger.dev v3 | Single job system; serverless-native; built-in observability |
| DocuSign | Docuseal | $0 self-hosted vs. per-envelope fees; same legal validity |
| GPT-4o + Claude 3.5 Sonnet routing | Claude 3.7 Sonnet (single model) | One API client; no routing logic; Claude 3.7 handles vision + long docs |
| scikit-learn delinquency model | Claude 3.7 Sonnet scoring | No training data required at launch; explainable output; simpler to operate |
| Doppler | Platform env vars (Vercel + Supabase) | Doppler is ops overhead, not a feature |
| Tamagui | NativeWind v4 | Tailwind tokens shared between web and mobile; single CSS system |
| Railway / Fly.io | Vercel (single platform) | One deployment target; no container management |

---

## 7. Revised Stack Summary

```
Frontend      Next.js 15 · Expo + Expo Router v3 · shadcn/ui · NativeWind v4
AI            Vercel AI SDK · Mastra · Claude 3.7 Sonnet · Claude 3.5 Haiku
Jobs          Trigger.dev v3
Data          Supabase (PostgreSQL + pgvector + RLS + Realtime + Storage)
Cache         Upstash Redis
Payments      Stripe Connect
Comms         Twilio
Screening     Checkr
Dispatch      TaskRabbit + Thumbtack
Locks         SmartLockAdapter (August / Yale / Schlage / Latch)
E-signature   Docuseal
Observability Langfuse + Sentry
Monorepo      Turborepo + pnpm workspaces
CI/CD         GitHub Actions → Vercel
```
