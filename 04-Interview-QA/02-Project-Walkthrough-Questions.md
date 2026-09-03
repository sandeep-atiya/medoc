# Project Walkthrough & Deep-Dive Questions (cross-project)

Interviewers ask these after your pitch. Each answer below uses real details from your code.

---

## A. Requirements & process

**Q1. How did you gather requirements?**
"From the actual users: call-center managers, QA auditors, sales heads. I would sit with them, look at the legacy screen or the manual process, and write a short spec. For QMS I wrote a 400-line data dictionary describing every field, its source table, and the rule — and I listed the assumptions that the business had to confirm (for example the New-to-Brand rule). For CallDropAutoDial the requirement was a written prompt from operations that I kept in the repo."

**Q2. How did you handle changing requirements?**
"By keeping business rules in config or constants and documenting them. Example: which dispositions trigger an Ameyo auto-dispose is an env list; the queue-to-campaign mapping is env; the repeat-order threshold is a named constant with a 'confirm with business' note."

**Q3. How did you prioritise when you were alone?**
"Production issues first, then features with the most users (agent CRM), then internal tools. I kept checklists (parity checklist, production checklist) so nothing was forgotten."

**Q4. How did you estimate?**
"By breaking a feature into the five backend files plus UI, and comparing with similar features I had already built. Code generation and the shared structure made estimates reliable."

---

## B. Architecture decisions

**Q5. Why the same layered pattern everywhere?**
"routes → validation → controller → service → repository. Validation stops bad input early, controllers stay thin, services hold business logic, repositories hold SQL. It made 45 reports and 47 CRM modules look the same, so bugs are easy to locate and a new developer can navigate quickly."

**Q6. Why raw SQL instead of an ORM for SQL Server/PostgreSQL?**
"Legacy schemas with hundreds of tables, complex reporting queries (window functions, DISTINCT ON, UNION ALL, temp tables), and performance control. Parameterised queries keep it safe. For document data (chat, leads) I used Mongoose because the shape fits."

**Q7. Why SQLite in three projects?**
"As a local read-model / cache: full SQL at local-disk speed, single file, no extra server. Used with WAL mode and a strict single-writer rule (one Node process, worker concurrency 1)."

**Q8. Why Express 5 and what changed from Express 4?**
"Native promise/async error handling (no need for express-async-errors), `req.query` became a getter (read-only) — which broke middlewares like xss-clean/hpp, so I stopped mounting them and wrote validated values to a custom property instead."

**Q9. When did you choose MongoDB?**
"WhatsApp messages and scraped leads — variable payloads, high write rate, simple access by contact/time. Compound indexes and aggregation pipelines give me the reports I need."

**Q10. How do the apps talk to each other?**
"Mostly through shared databases and iframe embedding: QMS and Patient-CRM are embedded in the ViewReports portal; Appointment-CRM is embedded in the Ameyo dialer. Each has its own API; no shared code, but the same structure."

---

## C. Data & performance

**Q11. Explain your ETL/sync design.**
"Registry of sources with a cursor each; the engine pulls chunks (`> cursor ORDER BY key`), sanitises, upserts by primary key in a transaction, records progress per source, checks a cancel flag between chunks, and reconciles stale jobs at startup. Runs inline on cron or in a separate worker via BullMQ."

**Q12. What is keyset pagination?**
"`WHERE id > @last ORDER BY id LIMIT n` — constant cost per page, unlike OFFSET which scans discarded rows. Used to mirror 4.5M order rows."

**Q13. What is a server-side cursor and when did you use it?**
"PostgreSQL `DECLARE cursor` + `FETCH 10000` — keeps the result set on the server; used to build a SQLite index from a 50M-row table without loading it into Node memory."

**Q14. Give one concrete optimisation with numbers.**
"QMS: a SQL Server view ran a per-row OUTER APPLY; bypassing the view with base tables + UNION ALL and adding indexes took a query from ~92 seconds to 200–350 ms."

**Q15. How do you cache and invalidate?**
"Three patterns: (1) key includes the data version — ViewReports keys on `last_sync_at`; (2) versioned key prefix — QMS bumps `v4 → v5` when the shape changes; (3) layered TTLs — Patient-CRM LRU 10 min, Redis 5 min, only cache non-empty successful results."

**Q16. How do you protect the server from heavy requests?**
"Date-range guard middleware before report routes, row-count check before exports (HTTP 413 above 800k rows), per-query timeouts with cancellation, rate limits sized for the office IP."

---

## D. Integrations

**Q17. How do webhooks work in your WhatsApp app?**
"Exotel POSTs `{ whatsapp: { messages: [...] } }` with a `callback_type` of `incoming_message` or `dlr`. I upsert by message SID so retries are idempotent, store the raw payload, update the contact, and emit Socket.IO events. Unknown types are stored for audit."

**Q18. How do you deal with unreliable third-party APIs?**
"Retry only on transient errors (network, 5xx, 429) with exponential backoff; never on 4xx. Log every attempt with HTTP code and body. Never let a messaging failure break the main transaction (booking is saved first). For Ameyo I live-tested and documented odd behaviours such as HTTP 200 with a `null` body on auth failure."

**Q19. How does the iframe SSO with Ameyo work?**
"Ameyo opens the CRM URL with user/session/campaign/extension, either as query params or via postMessage. The SPA stores them and sends `x-user-id`/`x-session-id` headers on every call; a backend middleware resolves the agent and caches for 5 minutes. The SPA's ProtectedRoute waits up to 2 seconds for the values before showing Unauthorized."

**Q20. How do you stream a call recording?**
"Backend rebuilds the upstream URL from a whitelisted base + validated ID (never from client input), forwards the `Range` header, relays `Content-Range`/`Accept-Ranges`, aborts upstream when the client disconnects, and sets a cross-origin resource policy just for that route."

---

## E. Delivery & operations

**Q21. How do you deploy and roll back?**
"Build with Vite in production mode; copy `dist` to the IIS site; copy backend, `npm ci --omit=dev`; iisnode restarts on watched files or I recycle the app pool; verify `/health/ready` and one real transaction. Roll back = redeploy the previous build folder (I keep the previous `dist`/backend folder)."

**Q22. How do you monitor?**
"Health endpoints (`/health`, `/health/status` shows loaded env files and feature flags, `/health/ready` pings all datastores → 503 when degraded), Winston daily-rotate logs, iisnode logs, PM2 logs, a periodic DB pool stats log."

**Q23. What happens when a database is down?**
"Depends on the app: ViewReports keeps serving SQLite-backed routes while external DBs reconnect; Patient-CRM treats SQL Server/Redis as optional; QMS returns 503 on `/ready` and degrades individual fields with warnings instead of failing the report."

**Q24. How did you document?**
"README with the core logic, runbooks (IIS deploy, production checklist), API references from live tests, Postman collections, OpenAPI YAML for the CRM, and comments that explain *why* (single process, no retry policy, watched files)."

**Q25. What would you do differently if you started again?**
"TypeScript from day one, tests per service, secrets outside the repo, CI/CD pipeline for IIS deploys, and a shared auth library across the embedded tools."

---

## F. Quick fire (one-line answers)

- **Node version used?** Node ≥18 (ESM modules, native fetch).
- **How many API endpoints in the CRM?** ~80 across 50 route files.
- **Largest table you worked with?** ~56M rows (call history), ~50M rows / ~39 GB (dialer call details).
- **Longest running job?** Full SQLite index build from PostgreSQL — minutes; runs in background with atomic file swap.
- **Rate limits?** Global 200/15 min (SaaS), auth 10/15 min, scrape 30/hour; 12,000/min for the office-shared IP in QMS.
- **Excel library?** ExcelJS. **Charts?** Recharts. **State?** Redux Toolkit / Zustand / TanStack Query depending on project.
- **Logging?** Winston (+ daily rotate) and Morgan.
- **Validation?** Joi for requests, envalid for environment.
- **Queues?** BullMQ (Email-Extract 4 queues, ViewReports sync worker).
- **Real-time?** Socket.IO in the WhatsApp dashboard.
