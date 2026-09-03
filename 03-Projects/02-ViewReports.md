# Project 2 — ViewReports (Call-Center BI & Reporting Platform)

**Repo:** `D:\ViewReports` (GitHub: ViewReports)
**Period:** June 2026 → August 2026 (75 commits)
**Size:** ~435 source files, ~59,000 lines (backend ~29k, frontend ~30k)
**My role:** Sole full-stack developer and architect — data model, ETL/sync engine, 45+ reports, RBAC + OTP auth, exports, admin panel, IIS deployment.

---

## 1. Pitches

**One line:**
> An enterprise reporting platform for a healthcare call center with 45+ reports over dialer, ACD and order data, built with a chunked ETL into a local SQLite read-model so reports on 50M+ row tables return in milliseconds.

**30-second pitch:**
> "ViewReports is the company's internal BI tool. It pulls call data from the Ameyo dialer warehouse in PostgreSQL and order/patient data from SQL Server, syncs it in chunks into a local SQLite read-model, and serves 45+ reports — transfer conversion, sales conversion, abandoned calls, agent and campaign performance, revenue, TV-channel DNIS attribution, trends, QMS audit and patient profile. It has JWT + email OTP login, per-report permissions from the user table, Excel/CSV export with row guards, and an admin panel that triggers and monitors data sync. Built with Express 5 and React 19 with TanStack Query/Table, deployed on IIS with iisnode."

**2-minute pitch (add):**
- Every report is the same 5 files: route → validation → controller → service → repository, so 45 reports stay maintainable.
- The sync engine has a registry of sources (ACD → dialer → call history → orders → daily summary → cohort orders), chunked incremental sync with cursors, progress tracking per source, cooperative cancellation and recovery of jobs stuck after a crash.
- Because better-sqlite3 is synchronous, I run a single Node process on IIS and added a `reportRangeGuard` middleware to cap date ranges so a huge query never freezes the process.
- A partial index for "failed association" reports is built in a **worker thread** so the API keeps serving during a multi-minute index build (query time 10 s → 4 ms).
- In-process report cache keyed by `report | last_sync_at | params`, which gives correct invalidation whenever the worker finishes a sync.
- Excel export limits: 800,000 rows Excel / 1,000,000 rows CSV, checked by a COUNT query before generating the file (HTTP 413 with a friendly message).

---

## 2. Business problem

- Management needed daily/weekly reports on call-center performance, but the data lived in two systems (PostgreSQL dialer warehouse with ~50M-row tables, and the SQL Server CRM).
- Old reports were slow (minutes) and often run manually as SQL by one person.
- Different users should see different reports (sales vs QA vs admin) and some should not be allowed to download.
- Data must be fresh (sync several times a day) but reports must be fast.

## 3. What I built — features

**Report groups (47 permission keys):**
- Transfer: transfer conversion, date-wise, agent-wise, unique transfer conversion/agent
- Sales: sales conversion, by agent, Hyderabad sales, doctor sales
- Abandoned: CS, NDR, appointment, digital, inbound
- Performance: overall, day-to-day, campaign-wise, Hyderabad, doctor
- Revenue: NTB vs repeat revenue, campaign revenue, doctor category revenue, order-call tracker
- Channels: channel conversion (with TV schedule Excel upload), show-wise, hourly-show, day DNIS, agent DNIS, DNIS performance
- Trends: daily trend, month trend, campaign transfer trend, transfer mapping
- Others: self hangup, failed association (campaign/agent)
- Quality: CRM vs dialer, call log (MIS), one-view
- CRM/QMS: patient profile lookup, QMS call audit (with Ameyo recording link)
- Analytics raw: raw ACD, raw dialer, transfer raw
- Admin: agent master CRUD, TV-DNIS CRUD, user permissions, data sync control, settings, cohort analytics (embedded from the Cohorts project)

**Auth & authorization:** login with legacy AES-encrypted password check → 6-digit OTP stored in Redis with TTL and emailed → verify OTP → 8-hour JWT. Per-report permissions from comma-separated strings on the user row; super-admin flag; `canViewReport(key)` / `canDownloadReport(key)` middlewares; sidebar filtered by the same keys.

**Data sync (ETL):** registry of sources, chunked incremental sync, per-source progress table, job history/stats, cancel, stale-job reconciliation. Runs inline (node-cron) or in a separate worker process (BullMQ repeatable job, concurrency 1). Also runnable from Windows Task Scheduler via `runSyncOnce`.

**Exports:** ExcelJS everywhere; CSV; guards on row counts.

**Frontend:** lazy-loaded routes, permission-filtered navigation, virtualised tables (TanStack Virtual), URL-synced filters (nuqs), live sync progress UI, Recharts charts, react-hook-form + zod forms, sonner toasts.

## 4. Architecture

```
SQL Server (CRM: patients, orders, users)      PostgreSQL (reportsdb: acd_call_details,
                                                dialer_call_details ~50M rows, call_history ~56M rows)
            └──────────────┬────────────────────────────┘
                           ▼
              Sync engine (src/sync: registry, engine, chunk, store, sanitize, sources/*)
              inline node-cron  OR  worker.js (BullMQ repeatable, concurrency 1)
                           ▼
              SQLite read-model (better-sqlite3, WAL, 15 tables, indexes, worker-thread index build)
                           ▼
    Express 5 API ── reportRangeGuard ── auth (JWT) ── reportPermission ── report routers (39)
                   ── in-process TTL+LRU cache keyed by last_sync_at ── ExcelJS export
                           ▼
    React 19 SPA (TanStack Query, Zustand auth store, permission-filtered sidebar, virtualised tables)
```

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 8 + React Compiler, react-router-dom 7 (lazy + Suspense), Zustand, TanStack Query 5 / Table 8 / Virtual, nuqs, Tailwind CSS 4, shadcn-style primitives (cva, clsx, tailwind-merge), lucide-react, cmdk, sonner, motion, Recharts 3, react-hook-form + zod, dayjs, react-day-picker, react-select, numeral, axios |
| Backend | Node.js ESM, Express 5, Joi (45 validation files), envalid, Winston + Morgan, helmet, cors, hpp, compression, express-rate-limit, jsonwebtoken, nodemailer, ExcelJS, multer (TV schedule upload), BullMQ, node-cron, uuid |
| Data | SQL Server (mssql 12), PostgreSQL (pg 8), SQLite (better-sqlite3 12, WAL), Redis (ioredis) |
| Deploy | IIS + iisnode (`nodeProcessCountPerApplication="1"`, `--max-old-space-size=4096`), SPA `web.config`, `.env.development/.staging/.production` |

## 6. Database design (SQLite read-model, 15 tables)

- `acd_call_details` (PK call_id; campaign, phone, dnis, call_type, answered/hungup, call_time, queue, wait time, agent, talk time, disposition… 6 indexes)
- `dialer_call_details` (PK call_id; crt_object_id, campaign, lead, phone, DID, system disposition, hangup cause, talk time, association type, agent, disposition class, transfer_to… 7 indexes)
- `call_history_details` (UNIQUE(call_id, udh_id) — one call → many agent legs; partial index for failed association)
- `order_details` (order no, patient, campaign, agent, amount, doctor, verifier, city/state, courier remark, order type…)
- `dialer_daily_summary` (day, campaign → calls, transfers, orders)
- `tv_channels` (channel ↔ DNIS)
- `sync_metadata`, `sync_jobs`, `sync_job_progress`, `source_stats`
- Cohort tables: `cohort_orders`, `cohort_summary`, `cohort_monthly_retention`, `cohort_dashboard_summary`, overall variants, `cohort_sync_meta`

Sources read live: SQL Server `AdminAuthentication`, `tblPatient`, `tblOrderDetails`, `CallActionDisposition`, `MainLinkUAD`; PostgreSQL `acd_call_details`, `dialer_call_details`, `call_history_details`, `agent_master`, `tv_channels`.

## 7. Notable technical implementations (explain any of these in depth)

1. **Chunked incremental sync** — each source defines a cursor (date/id), the engine pulls chunks, sanitises rows, writes in SQLite transactions, records progress; cancellation is checked between chunks; stale "running" jobs are reconciled at startup.
2. **Single-writer rule** — SQLite allows one writer; therefore IIS runs one Node process and BullMQ worker concurrency is 1 with a 10-minute lock.
3. **Worker-thread index build** — `ensureFaIndexInBackground()` builds a partial index off the event loop.
4. **Leg pairing without an index** — QMS audit pairs call legs by `crt_object_id` within a bounded `call_time` window (measured: ~7k multi-leg interactions/day, max gap 51 min) instead of adding a costly index.
5. **Report range guard** — middleware caps the date range before any report runs.
6. **Cache with data-stamp key** — `name | last_sync_at | params` so cache is naturally invalidated after every sync even across processes.
7. **Resilient process** — operational errors (ECONNRESET, ETIMEDOUT, EPIPE, ENOTFOUND) are logged, not fatal, so an IIS recycle never kills an in-flight sync; HTTP listens immediately while external DBs connect in parallel.
8. **Export guards** — COUNT first; 413 if above 800k (Excel) / 1M (CSV).
9. **OTP login** — password check compatible with the legacy ASP.NET encryption (PBKDF2 → AES-256-CBC, UTF-16LE), then OTP in Redis with TTL.

## 8. Deployment

- `backend/web.config`: iisnode handler, rewrite-all rule, `node_env=production`, `nodeProcessCountPerApplication="1"` (deliberate), `nodeProcessCommandLine="node.exe --max-old-space-size=4096"`, `maxConcurrentRequestsPerProcess=4096`, iisnode logs.
- `server.js` accepts iisnode's named-pipe PORT; `trust proxy` enabled.
- Frontend `web.config`: SPA rewrite + MIME maps (svg, woff2, json).
- Sync can run inline, as a worker (NSSM Windows service), or via Windows Task Scheduler (`npm run sync:once:prod`).
- Environment files for dev/staging/prod; CORS allowlist for the office hosts.

## 9. Challenges and solutions (STAR)

1. **Reports on 50M+ rows took minutes.** → Built the SQLite read-model + incremental sync; reports now hit local indexed tables. Result: sub-second for most reports.
2. **A heavy report froze the whole API** (synchronous SQLite in a single process). → Added `reportRangeGuard`, in-process cache, and moved long syncs to a separate worker process. Result: no more app-pool recycles during reports.
3. **Index build took minutes and blocked startup.** → Built it in a worker thread on first need; API serves meanwhile.
4. **Users exporting millions of rows crashed memory.** → COUNT guard → 413 with a clear message; encourage narrower filters.
5. **Different users need different reports.** → Permission keys in the user table, backend middleware + frontend sidebar both read the same key list.
6. **Sync job stuck as "running" after a server restart.** → `reconcileStaleJobs` on boot + cooperative cancel flag.

## 10. Results / impact

- 45+ reports available self-service to managers instead of manual SQL.
- Report time from minutes to milliseconds/seconds on cached data.
- Controlled downloads through permissions; OTP-secured login.
- Reusable sync engine and QMS/cohort modules embedded in one portal.

## 11. Interview questions specific to this project

**Q: Why SQLite as a cache instead of Redis or materialised views?**
A: Reports need SQL (joins, group by, windows) over tens of millions of rows; Redis is key-value. Materialised views on the source DBs would load the production dialer DB. A local SQLite file with indexes gives full SQL at local-disk speed, zero network hops, and it is a single file I can rebuild anytime.

**Q: What is the risk with SQLite and how did you handle it?**
A: One writer at a time and synchronous calls. I run one Node process, one sync worker with concurrency 1, WAL mode for readers, and cap query ranges.

**Q: How does the incremental sync know where to continue?**
A: Each source keeps a cursor (last synced time/id) in `sync_job_progress`/`sync_metadata`; chunks pull `> cursor` ordered ascending; rows are upserted by primary key.

**Q: How is cache invalidation handled?**
A: The cache key includes `last_sync_at`. When a sync finishes, the stamp changes, so old entries are simply never hit again and expire by TTL/LRU.

**Q: How did you implement RBAC?**
A: Permission strings on the user row (view list, download list, super-admin flag). A middleware factory `canViewReport(key)` checks the key per route; the same key list drives the sidebar.

**Q: Why OTP by email?**
A: Business required two-factor for financial reports; email OTP needed no extra vendor; OTP stored in Redis with a short TTL, one-time use.

**Q: How do you keep 45 reports consistent?**
A: A strict 5-file pattern per report, shared helpers for date ranges, pagination, export, and shared validation schemas. New report = copy the pattern.

**Q: How do you handle huge exports?**
A: COUNT before generating; stream rows into ExcelJS; enforce limits; return 413 with a message.

**Q: How did you test performance?**
A: Measured with real data volumes (e.g., 6,941 multi-leg interactions/day, max gap 51 minutes) and used those numbers to choose the leg-pairing window instead of adding an index.

## 12. Be careful — what NOT to claim

- No Socket.IO / SSE; the UI **polls** the sync status endpoint. Say "polling".
- `fast-csv`, `swagger-jsdoc`, `swagger-ui-express` are installed but unused; there is **no Swagger UI** here (Swagger exists in Appointment-CRM).
- Ameyo recordings are downloaded **by the browser** using the user's Ameyo session; the API only builds the URL (the streaming proxy is in the QMS project).
- Credentials are committed in env files — if asked about secrets, present the improvement plan.
- `docs/SYNC_WORKER.md` is referenced in code but does not exist; do not quote it.

## 13. What I would improve next

- Server-Sent Events for sync progress instead of polling.
- Move the read-model to PostgreSQL/ClickHouse if concurrency grows beyond one writer.
- Add automated tests for repositories with a seeded SQLite fixture.
- Secret manager + CI/CD pipeline for IIS deployment.

## 14. Resume bullets

- Architected and built an enterprise BI platform (Express 5 + React 19) serving 45+ call-center reports from SQL Server and PostgreSQL sources (50M+ row tables) via a chunked incremental ETL into a SQLite read-model, cutting report time from minutes to sub-second.
- Implemented JWT + email-OTP two-factor authentication, per-report RBAC, ExcelJS/CSV exports with 800k/1M-row guards, and an admin console for sync control with progress tracking, cancellation and crash recovery.
- Engineered for a single-process IIS host: worker-thread index builds, report range guards, data-stamp keyed caching, resilient error handling, and a BullMQ worker with single-writer locking.

## 15. Keywords

React 19, TanStack Query, TanStack Table, virtualisation, Zustand, Tailwind 4, Recharts, Vite, Node.js, Express 5, ETL, data sync, SQLite, better-sqlite3, PostgreSQL, SQL Server, Redis, BullMQ, node-cron, worker threads, JWT, OTP, 2FA, RBAC, ExcelJS, IIS, iisnode, BI, reporting, performance optimisation
