# Quick Revision Cheat Sheet — read 30 minutes before the interview

---

## Your one-liner
"Full-stack Node/React developer; built and deployed 8 production apps end-to-end for a healthcare call-center business on SQL Server, PostgreSQL, MongoDB, SQLite and Redis; hosted on IIS and Cloudways."

## The 8 apps (say in this order)
1. **Appointment-CRM** — legacy ASP.NET → React + Express 5 + SQL Server; 47 modules; 463 generated models; Ameyo/WhatsApp/SMS; IIS.
2. **ViewReports** — 45+ reports; ETL → SQLite read-model; JWT+OTP; RBAC; Excel guards; IIS.
3. **WhatsApp Dashboard** — Exotel webhooks → MongoDB → Socket.IO; multi-number routing; Cloudways/PM2/Apache.
4. **QMS** — Call ID → two-leg audit + streamed recording; 92 s → 300 ms.
5. **Patient-CRM** — 4 phone numbers; LRU → Redis → SQLite → PostgreSQL; 2 min → ms.
6. **Cohorts-Analytics** — Python port; 4.5M rows keyset-mirrored; heatmap; styled Excel; HMAC OTP.
7. **CallDropAutoDial** — cron 30 min; DISTINCT ON + NOT EXISTS; DRY_RUN; 1 process.
8. **Email-Extract** — BullMQ 4 queues; Cheerio→Puppeteer; MX validation; JWT refresh (MVP).

## Numbers
8 apps · ~155k LOC · 9 months · 463 models · 47 modules · 45+ reports · 50M+ rows · 92 s→300 ms · 2 min→ms · 800k/1M export guards · 6 on IIS + 1 on Cloudways

## Architecture sentence
"routes → Joi validation → controller → service → repository; helmet/cors/rate-limit/auth/error middleware; Winston logs; health/ready endpoints; env validated with envalid."

## Hero stories (S-T-A-R in 4 lines each)
- **ViewReports freeze:** sync SQLite in 1 process → range guard + worker-thread index + separate sync worker + stamp-keyed cache → no recycles, ms reports.
- **QMS 92 s:** per-row OUTER APPLY view → base tables + UNION ALL + indexes + timeouts with cancel → 300 ms.
- **CallDrop empty window:** child table filled hourly → switched to live parent table → correct lists; documented.
- **iisnode old code:** watchedFiles missing src → fixed + flag logging + restart command.
- **WhatsApp voice notes rejected:** Cloudinary transcode to MP3 → accepted.

## Tech one-liners
- **Express 5:** async errors native; `req.query` read-only getter.
- **iisnode PORT:** named pipe; never parseInt.
- **SQLite rule:** one writer → one process, worker concurrency 1, WAL.
- **Keyset pagination:** `WHERE id > last ORDER BY id LIMIT n`.
- **DISTINCT ON:** first row per group in PostgreSQL.
- **Cursor:** `DECLARE … CURSOR; FETCH 10000`.
- **Cache keys:** TTL + version prefix + data stamp; cache only non-empty success.
- **Retry:** only network/5xx/429; exponential `250·2^(n-1)`; never 4xx; no retry where duplicate side-effects are dangerous.
- **Webhook:** public route before auth; idempotent upsert by provider id; store raw.
- **OTP:** Redis with TTL (ViewReports) or HMAC-in-JWT stateless (Cohorts), constant-time compare.
- **RBAC:** permission keys on user row → middleware per route → same keys filter sidebar.
- **Socket.IO scaling:** Redis adapter + sticky sessions; fork mode today.
- **BullMQ:** attempts, backoff, concurrency, repeatable jobs, removeOnComplete.
- **Range streaming:** forward `Range`, relay `Content-Range`, 206, abort on disconnect, whitelist upstream.

## Honesty flags (do not overclaim)
- CRM: socket.io/bull/multer unused; cron stubs; header SSO for agents, JWT for admin.
- ViewReports: polling not sockets; no Swagger here.
- WhatsApp: hardcoded UI login + API key; templates 501; no webhook signature.
- Cohorts: no Postgres/Redis/BullMQ; cohort routes not JWT-protected.
- QMS / Patient-CRM: no auth inside (host portal); archive recordings switchable.
- CallDrop: no auth, no retries by design.
- Email-Extract: MVP, export UI not wired, known bugs in backlog.
- Secrets in some env files → improvement plan.

## Improvement plan (say when asked "what next")
TypeScript · tests per service · secrets out of repo · CI/CD · shared JWT for embedded tools · Socket.IO Redis adapter · SSE for progress · Docker for the SaaS stack

## Questions to ask them
Deployment pipeline? Team/code review process? Biggest technical challenge next 6 months? Success in 90 days?
