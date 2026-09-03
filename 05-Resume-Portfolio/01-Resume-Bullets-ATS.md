# Resume Content — ATS-friendly bullets (copy-paste ready)

Format: Action verb + what you built + tech + measurable result. Keep 3–5 bullets per project; pick the projects that match the job.

---

## Professional summary (choose one)

**Version A (general full-stack):**
Full-Stack Developer (Node.js, Express, React) with hands-on production experience building and deploying 8 business applications end-to-end — call-center CRM, BI reporting platform, real-time WhatsApp inbox, analytics dashboards and automation services — on SQL Server, PostgreSQL, MongoDB, SQLite and Redis. Strong in REST API design, ETL/caching for large datasets (50M+ rows), third-party integrations (dialer, WhatsApp, SMS, courier APIs), and deployment on Windows IIS (iisnode) and Linux (PM2, Apache). Owns the full lifecycle from requirements to production support.

**Version B (backend-leaning):**
Backend-focused Full-Stack Engineer specialising in Node.js/Express APIs over SQL Server, PostgreSQL and MongoDB, with React front-ends. Built layered, validated, documented services (Joi, OpenAPI, Winston), background jobs and queues (node-cron, BullMQ), real-time messaging (Socket.IO), and read-model/caching strategies that cut report times from minutes to milliseconds. Deployed and operated 7 production systems on IIS and Cloudways.

---

## Experience section — role header

**Full-Stack Developer** — [Company], [City] — [Month Year] – Present
Client domain: healthcare tele-consultation & call-center operations (Ameyo dialer, clinics, e-commerce orders)

---

## Project bullets

### Appointment-CRM — Call-Center CRM (React 19, Node/Express 5, SQL Server, Redis, IIS)
- Re-engineered a legacy ASP.NET WebForms call-center CRM into a React 19 + Express 5 application on the existing SQL Server databases (two connection pools, 460+ tables), preserving functional parity for daily agent use inside the Ameyo dialer.
- Wrote a SQL-DDL-to-code generator producing 463 model classes plus controller/route scaffolds; delivered 47 API modules (~80 endpoints) with Joi validation, OpenAPI/Swagger docs, RBAC, audit trails and health/readiness probes.
- Integrated Ameyo dialer APIs (dial, dispose, hangup, iframe SSO), Exotel WhatsApp Business API (templated confirmations with retry/backoff and delivery-status webhooks), ValueFirst SMS and courier tracking APIs (Delhivery, XpressBees, ShipDelight).
- Built 23 agent disposition panels and a dynamic consultation module driven by 16 JSON form schemas with draft autosave (Redux Toolkit, redux-persist, Tailwind, Recharts).
- Deployed on Windows Server IIS with iisnode (multi-process, AlwaysRunning app pool, rotating Winston logs) and supported the system in production.

### ViewReports — Call-Center BI Platform (React 19, Express 5, SQL Server, PostgreSQL, SQLite, Redis, BullMQ, IIS)
- Architected a reporting platform serving 45+ reports over 50M+ row telephony and order tables via a chunked incremental ETL into a SQLite read-model, reducing report time from minutes to sub-second.
- Implemented JWT + email-OTP two-factor login (legacy-compatible password check), per-report RBAC from user permission strings, and an admin console for data sync with progress, cancellation and crash recovery (BullMQ worker, node-cron fallback).
- Engineered for a single-process host: worker-thread index builds (10 s → 4 ms queries), report date-range guards, data-stamp keyed caching, resilient error handling; ExcelJS/CSV exports with 800k/1M-row guards.
- Built the React front-end with TanStack Query/Table/Virtual, Zustand, Tailwind 4, lazy routes and a permission-filtered navigation of 47 report keys.

### WhatsApp Dashboard — Real-Time Business Inbox (MERN, Socket.IO, Exotel API, Cloudinary, Cloudways)
- Built a real-time WhatsApp inbox on the Exotel WhatsApp Business API: inbound/DLR webhooks with idempotent upserts, automatic multi-business-number routing, unread tracking with compound MongoDB indexes, media via Cloudinary (audio transcoding), and aggregation-based reports.
- Delivered live updates with Socket.IO (inbound, outbound status, unread events) and a React chat UI (emoji, attachments, media bubbles); implemented exponential-backoff retries for provider calls.
- Deployed on Cloudways (Linux) with PM2 and an Apache reverse proxy including WebSocket upgrade; authored deployment/check/test scripts, Postman collections and 17 operational guides.

### QMS — Call-Quality Audit Report (React 19, Express 5, PostgreSQL, SQL Server, Redis, Ameyo, IIS)
- Designed a call-quality audit tool that reconstructs multi-leg dialer interactions from PostgreSQL, enriches them with clinical data from SQL Server, and streams Ameyo call recordings through a Range-aware, whitelist-hardened proxy for in-page playback.
- Optimised a critical SQL Server query from ~92 s to ~300 ms by bypassing a per-row OUTER APPLY view and adding indexes; added per-query timeouts with cancellation, versioned Redis caching and a never-500 degradation contract with user-visible warnings.

### Patient-CRM — Cached Call-History Lookup (React 19, Express 5, PostgreSQL, SQL Server, SQLite, Redis, IIS)
- Built a lookup service resolving a patient's four phone numbers and returning 30-day call history from a 50M-row warehouse through a three-tier cache (in-process LRU → Redis → SQLite index → PostgreSQL), cutting search time from ~2 minutes to milliseconds.
- Implemented a memory-safe ETL using PostgreSQL server-side cursors with atomic SQLite file swaps and 15-minute incremental syncs; hardened for iisnode and embedded as an iframe tool in the dialer.

### Cohorts-Analytics — Retention & LTV Dashboard (React 19, Express 5, SQL Server, SQLite, node-cron, ExcelJS, IIS)
- Ported a Python cohort-analysis tool to a web app: mirrored 4.5M SQL Server order rows into SQLite via keyset pagination, implemented NTB/overall cohort retention and LTV SQL, and delivered a heatmap dashboard with CSV and pixel-matched styled Excel exports; validated against the original outputs.
- Added stateless email-OTP two-factor login and deployed on IIS with iisnode and an ARR reverse proxy.

### CallDropAutoDial — Lead-Recovery Automation (Node/Express 5, PostgreSQL, node-cron, Ameyo API, IIS)
- Built and deployed a scheduled service that detects dropped/abandoned inbound calls every 30 minutes, de-duplicates and normalises numbers, and batch-uploads them to the correct Ameyo outbound campaigns via the uploadContacts API.
- Engineered for safe automation: DRY_RUN mode, re-entrancy guard, reconnect-exclusion SQL, single-process IIS deployment, envalid fail-fast config, structured logging, and a live-verified API reference plus IIS runbook.

### Email-Extract — Lead-Generation SaaS Prototype (React 19, Express 4, MongoDB, Redis, BullMQ, Puppeteer)
- Built a four-stage BullMQ pipeline (SerpAPI search → Cheerio/Puppeteer scraping → email validation with disposable-domain and DNS MX checks → storage) with per-queue retry/backoff, worker concurrency and multi-layer de-duplication.
- Implemented JWT access/refresh authentication, owner-scoped data access, tiered rate limiting, Winston logging with centralised error mapping, and a Redux Toolkit + React Query dashboard with Recharts.

---

## One-line project list (for a compact resume)

- **Appointment-CRM** — Legacy ASP.NET → React/Node call-center CRM; 47 modules; Ameyo, WhatsApp, SMS integrations; IIS.
- **ViewReports** — 45+ report BI platform; ETL to SQLite read-model; JWT+OTP; RBAC; Excel export; IIS.
- **WhatsApp Dashboard** — Real-time Exotel WhatsApp inbox; Socket.IO; MongoDB; Cloudways/PM2.
- **QMS** — Call-quality audit with recording streaming; 92 s → 300 ms query optimisation.
- **Patient-CRM** — 3-tier cached call-history lookup over 50M-row warehouse.
- **Cohorts-Analytics** — Cohort retention/LTV dashboard with styled Excel export.
- **CallDropAutoDial** — Cron-based dropped-call recovery into Ameyo campaigns.
- **Email-Extract** — BullMQ scraping/validation pipeline SaaS prototype.

---

## Achievements line (optional "Key achievements" section)

- Delivered 8 production/prototype applications (~155k LOC) end-to-end in ~9 months.
- Reduced reporting latency from minutes to milliseconds on 50M+ row datasets.
- Cut a critical query from 92 s to ~300 ms; a lookup from ~2 min to ms.
- Automated appointment confirmations (WhatsApp + SMS) and dropped-call recovery.
