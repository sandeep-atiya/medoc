# System Design & Architecture Questions — with designs you actually built

Interviewers ask "design X". Use your real systems as the base and extend them.

---

## 1. "Design a reporting system over a huge call database" (you built ViewReports)

**Requirements:** 45+ reports, 50M+ rows, many users, fast, exportable, permissioned, fresh within a few hours.

**Design you gave:**
1. **Sources:** SQL Server (CRM) + PostgreSQL (dialer warehouse). Read-only.
2. **ETL/sync:** registry of sources with cursors; chunked incremental pulls; sanitise; upsert into a **read-model** (SQLite locally; PostgreSQL/ClickHouse at larger scale); progress + cancel + crash recovery; run in a separate worker (BullMQ) with single-writer lock.
3. **API:** Express, one 5-file module per report; validation; **range guard** to cap heavy queries; **cache** keyed by `report|last_sync_at|params` so invalidation is automatic after every sync.
4. **Exports:** COUNT first; hard limits (800k Excel / 1M CSV) → 413; stream rows.
5. **Auth:** JWT + OTP; per-report permission keys shared by API and sidebar.
6. **Ops:** one process for SQLite; worker-thread index builds; health/ready; logs.

**Scale-up answer:** move the read-model to PostgreSQL/ClickHouse with partitioning by month, pre-aggregate daily summaries (you already have `dialer_daily_summary`), add Redis for hot report cache across processes, push exports to a queue and email a link, add SSE for sync progress.

---

## 2. "Design a WhatsApp / chat inbox for agents" (you built it)

1. **Ingress:** provider webhooks (inbound + delivery receipts) → public endpoint → validate → **idempotent upsert by provider message id** → store raw payload.
2. **Storage:** MongoDB `Message` (from, to, direction, status, isRead, businessNumberId, timestamp; compound indexes for chat list and unread counts), `Contact` (lastBusinessNumberId), `Media`.
3. **Real-time:** Socket.IO emits `message:inbound`, `message:outbound` (status updates), `message:unread`, `contact:upsert`.
4. **Outbound:** API → provider with retry/backoff on transient errors; message saved `queued` first; DLR updates status.
5. **Routing:** reply from the same business number the customer used.
6. **Media:** upload to Cloudinary; transcode audio to MP3.
7. **Scale:** PM2 cluster + Socket.IO Redis adapter + sticky sessions; queue outbound sends (BullMQ); per-agent assignment of conversations; read receipts; search index.

---

## 3. "Design a scraping / lead-generation pipeline" (you built Email-Extract)

1. API accepts a query → creates `Search` → enqueue **serp** job.
2. **serp worker** (concurrency 2) → SerpAPI pages → fan-out one **scraper** job per URL.
3. **scraper worker** (concurrency 5, 3 attempts, exponential backoff) → Cheerio → contact pages → Puppeteer fallback → extract emails → enqueue **email** job.
4. **email worker** (concurrency 3) → validate (regex → disposable → MX) → dedup (Set → DB `$in` → unique index) → `insertMany unordered` → update counters.
5. **export worker** → CSV/Excel/PDF → file → notify.
6. Rate limits per user, credits, ownership scoping.
**Scale:** proxy pool, per-domain politeness, browser pool, DLQ for failed jobs, metrics (Bull board), horizontal workers.

---

## 4. "Design a cache for a slow lookup" (you built Patient-CRM)

Tiered: L1 in-process LRU (fast, per process) → L2 Redis (shared, TTL) → L3 local pre-built index (SQLite refreshed every 15 min from the warehouse via server-side cursor, atomic file swap) → source DB (fallback + gap-fill for the newest rows). Rules: cache only successful non-empty results; distinct timeouts per tier; every tier optional and non-fatal.

---

## 5. "Design a scheduled job that must never run twice" (you built CallDropAutoDial)

- One process (IIS worker count 1) + in-process running flag → skip overlapping cycles.
- Idempotent selection: `DISTINCT ON` + normalised dedup + reconnect exclusion window.
- `DRY_RUN` flag for safe rollout; manual trigger endpoint; preview endpoint.
- Log-only failure policy (agreed) with full payload logging; per-record accounting.
- Scale-up: persist an "uploaded today" ledger; distributed lock in Redis if multiple instances; alerting.

---

## 6. "Design a legacy migration strategy" (Appointment-CRM)

- Keep the database; build a new API + SPA beside the old app; parity checklist; mirror legacy endpoints under `/legacy/*`; generate models from DDL; feature-by-feature cutover inside the same host (dialer iframe); compare outputs; keep old app as reference until parity is signed off.

---

## 7. "Design a call-quality audit tool" (QMS)

- Input Call ID → resolve interaction id → gather all legs → classify original/transferred → map fields → enrich from CRM DB with per-query timeouts → probe recording servers (live then archive) with ranged GET → stream via hardened proxy → cache with versioned key → never-500 with warnings.

---

## General system-design questions

**Q: Monolith vs microservices — what did you do?**
"Modular monoliths per product (each app is its own deployable with a layered structure) sharing databases. That fit a solo developer and IIS hosting. With a team I would extract shared auth and a shared reporting read-model service first."

**Q: How do you make an API scalable?**
Stateless handlers (JWT), connection pooling, caching with correct invalidation, pagination (keyset), async work in queues, horizontal processes (PM2 cluster / IIS multiple processes when stateless), rate limits.

**Q: How do you handle failures of dependencies?**
Timeouts everywhere, retries only on transient errors, circuit-breaker style flags (archive recordings disabled by env when slow), degrade with warnings instead of failing, readiness endpoint reports the state.

**Q: How do you design for observability?**
Structured logs with request ids, health/ready/status endpoints, startup logs of flags, job summaries as JSON, DB pool stats.

**Q: How do you choose a database?**
Relational when data is structured and joins/reports matter (SQL Server/PostgreSQL); document DB for flexible payloads and simple access paths (MongoDB); SQLite as an embedded read-model; Redis for ephemeral/cached/queue data.

**Q: CAP / consistency for your read-models?**
Read-models are eventually consistent (minutes behind); acceptable for reports; cache keys include the sync stamp so users never see mixed data.

**Q: How would you add multi-tenancy?**
Tenant id on every row/document + scoped queries (as done with `owner`), per-tenant rate limits and credits, separate DBs for large tenants.

---

## Diagrams to draw on a whiteboard (practise)

1. Layered Express module (5 boxes in a row).
2. ETL → read-model → API → SPA (ViewReports).
3. Webhook → DB → Socket.IO → browsers (WhatsApp).
4. Tiered cache (Patient-CRM).
5. Queue pipeline with workers (Email-Extract).
6. IIS site → iisnode → Node processes; ARR proxy `/api`.
