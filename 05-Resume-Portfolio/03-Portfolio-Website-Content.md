# Portfolio Website Content — Hero, About, Project Cards, Case Studies

Written safely (no client names, IPs or secrets). Replace [brackets].

---

## Hero

**[Your Name]**
Full-Stack Developer · Node.js · React · SQL & NoSQL
*I build and ship business applications end-to-end — from requirements to production.*
[View Projects] [Download Resume] [GitHub] [LinkedIn]

---

## About (short)

I'm a full-stack developer who has built and deployed eight applications for a healthcare tele-consultation company running a large call center. My work covers a call-center CRM that replaced a legacy ASP.NET system, an enterprise BI platform with 45+ reports over 50-million-row datasets, a real-time WhatsApp inbox, analytics dashboards, and automation services. I work across React, Node.js/Express, SQL Server, PostgreSQL, MongoDB, SQLite and Redis, and I deploy on Windows IIS and Linux (PM2/Apache). I like turning vague business problems into precise, documented, fast systems.

---

## Project cards (title · one-liner · stack tags · 3 highlights)

### 1. Call-Center CRM (Legacy Modernisation)
Agent-facing CRM inside a dialer, replacing an ASP.NET WebForms app.
`React 19` `Redux Toolkit` `Express 5` `SQL Server` `Redis` `IIS`
- 463 models generated from SQL DDL; 47 API modules with OpenAPI docs
- Dialer, WhatsApp (Exotel) and SMS integrations with retry & delivery tracking
- Zero-click iframe SSO; 23 disposition panels; 16 dynamic consultation forms

### 2. Call-Center BI Platform
45+ self-service reports over telephony and order data.
`React 19` `TanStack` `Express 5` `PostgreSQL` `SQL Server` `SQLite` `Redis` `BullMQ` `IIS`
- Chunked ETL into a SQLite read-model: minutes → milliseconds
- JWT + email OTP, per-report RBAC, Excel/CSV export guards
- Worker-thread index builds, data-stamp caching, sync console with cancel/recovery

### 3. Real-Time WhatsApp Inbox
Shared inbox for two WhatsApp Business numbers via Exotel.
`MERN` `Socket.IO` `MongoDB Atlas` `Cloudinary` `PM2` `Apache` `Cloudways`
- Idempotent webhooks, delivery-status tracking, unread badges
- Automatic reply-from-the-right-number routing
- Media with audio transcoding; aggregation reports

### 4. Call-Quality Audit (QMS)
One Call ID → full audit sheet with playable recording.
`React 19` `Express 5` `PostgreSQL` `SQL Server` `Redis` `IIS`
- Two-leg call reconstruction algorithm
- Range-aware, whitelist-hardened audio streaming proxy
- 92 s → 300 ms query optimisation; never-500 degradation with warnings

### 5. Patient Call-History Lookup
Search any phone/patient ID → 30-day history across all numbers.
`React 19` `Express 5` `PostgreSQL` `SQL Server` `SQLite` `Redis` `IIS`
- Three-tier cache (LRU → Redis → SQLite → PostgreSQL)
- Server-side cursor ETL with atomic file swap; 15-min incremental sync
- ~2 min → milliseconds

### 6. Cohort Retention & LTV Dashboard
Monthly cohort heatmaps, KPIs and styled Excel export.
`React 19` `Express 5` `SQL Server` `SQLite` `node-cron` `ExcelJS` `IIS`
- Ported from Python; validated to the same numbers
- 4.5M rows mirrored via keyset pagination
- Stateless HMAC OTP login; ARR reverse proxy

### 7. Dropped-Call Recovery Automation
Every 30 minutes: find dropped/abandoned callers → push to outbound campaigns.
`Node.js` `Express 5` `PostgreSQL` `node-cron` `Ameyo API` `IIS`
- DISTINCT ON + reconnect-exclusion SQL; two-stage dedup
- DRY_RUN, re-entrancy guard, single-process deployment
- Live-verified API reference and runbook

### 8. Lead-Generation Pipeline (Prototype)
Search → scrape → validate business emails.
`React 19` `Express 4` `MongoDB` `Redis` `BullMQ` `Puppeteer` `Cheerio`
- Four-stage queue pipeline with retry/backoff and concurrency
- Cheerio-first, Puppeteer fallback; disposable-domain + MX validation
- JWT access/refresh, owner-scoped data, rate limiting

---

## Case study template (use for the top 2–3 projects)

**Title** — one line
**Context** — who used it, what was broken (2–3 sentences)
**Constraints** — legacy DB, on-prem Windows, single developer, data volume
**Architecture** — diagram (copy from the project file) + 4–6 bullets
**Key decisions** — why SQLite read-model / why raw SQL / why header SSO
**Hard problem** — one story with numbers (92 s → 300 ms; minutes → ms)
**Outcome** — what changed for users
**What I'd do next** — 3 bullets

---

## Testimonial / proof ideas (optional)

- Screenshots with client data blurred (dashboards, chat UI, report heatmap).
- Short screen recordings of the audit tool playing a recording (use test data).
- Architecture diagrams exported as images.
- A "by the numbers" strip: 8 apps · 155k LOC · 4 DB engines · 45+ reports · 50M+ rows.
