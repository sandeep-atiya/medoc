# My Roles and Responsibilities — Full-Stack Developer (End-to-End Owner)

Use this file when the interviewer asks: *"What was your role?"*, *"What were your responsibilities?"*, *"Did you build it alone?"*, *"Walk me through your development process."*

---

## 1. One-paragraph answer (memorise)

> "I worked as a full-stack developer and was the single owner of these applications from start to finish. I collected requirements directly from business users and call-center managers, designed the database access and API structure, built the backend in Node.js/Express and the frontend in React, integrated third-party systems like the Ameyo dialer, Exotel WhatsApp and SMS gateways, wrote the deployment configuration, deployed to Windows IIS with iisnode and to Cloudways with PM2, and then supported the apps in production — fixing live issues, tuning performance and adding features."

---

## 2. Responsibilities by phase of the software lifecycle

### Phase 1 — Requirement gathering & analysis
- Talked with call-center managers, QA auditors, sales heads and IT to understand the real problem (e.g., QA auditors were manually cross-checking the dialer console against two databases for every call).
- Read the **legacy ASP.NET WebForms code** (C# code-behind, App_Code classes, SQL) to understand existing business rules before rewriting them.
- Wrote/maintained requirement notes and specs (example: a 400+ line build spec and data dictionary for the QMS report, a business-rule prompt file for CallDropAutoDial, a parity checklist for Appointment-CRM).
- Identified and confirmed unclear business rules with stakeholders and documented the assumptions (e.g., "New-to-Brand vs Repeat customer" rule, which call leg decides "Hangup By").

### Phase 2 — Architecture & design
- Chose the stack per project: Express 5 + React 19 as the standard; MongoDB where data is document-shaped (chat messages, scraped leads); SQLite read-models where reports had to be fast on 50M+ row tables.
- Designed the **layered backend structure** (routes → validation → controller → service → repository) and reused it across every project.
- Designed **data flows**: ETL sync from SQL Server / PostgreSQL to SQLite; 3-tier cache (in-memory LRU → Redis → SQLite → PostgreSQL); BullMQ queue pipelines.
- Designed **API contracts** (REST, OpenAPI/Swagger for Appointment-CRM), consistent `ApiResponse` / `ApiError` shapes, health/readiness endpoints.
- Planned **security**: JWT + bcrypt, email OTP two-factor, permission strings for per-report RBAC, API-key for machine-to-machine, phone-number masking for privacy.

### Phase 3 — Database work
- Wrote **raw parameterised SQL** for SQL Server (mssql/tedious) and PostgreSQL (pg); no ORM in the enterprise apps for full control of performance.
- Designed **Mongoose schemas** with compound indexes for MongoDB apps.
- **Index design and query optimisation** (bypassing a slow SQL Server view: ~92 s → ~300 ms; partial indexes in PostgreSQL/SQLite; keyset pagination for large syncs).
- Wrote **SQL setup scripts** (DDL, index scripts marked idempotent for production go-live).
- Wrote a **code generator** that reads SQL DDL and produces 463 model classes plus controller/route scaffolds.
- Handled **SQL Server dynamic data masking** (UNMASK permission) and legacy **ASP.NET password encryption** (PBKDF2 + AES-256-CBC, UTF-16LE) so the existing user table kept working.

### Phase 4 — Backend development
- Built REST APIs in Express with Joi validation, centralised error handling, Winston logging, Helmet/CORS/rate limiting.
- Built **integrations**: Ameyo dialer REST commands (dial, dispose, hangup, uploadContacts, voice-log download), Exotel WhatsApp (send template messages, delivery-status webhooks), ValueFirst SMS, courier tracking (Delhivery, XpressBees, ShipDelight), Cloudinary uploads, SerpAPI.
- Built **background jobs**: node-cron schedulers with timezone, BullMQ queues/workers with retry & backoff, single-writer guards, re-entrancy guards, DRY_RUN safety flags.
- Built **real-time** features with Socket.IO (inbound message, unread count, delivery status events).
- Built **exports**: ExcelJS styled sheets (frozen panes, colour heatmap), CSV with UTF-8 BOM, row-count guards that return HTTP 413 before generating huge files.
- Built an **audio streaming proxy** with HTTP Range support so call recordings play inside the page.

### Phase 5 — Frontend development
- Built SPAs in React 19 with Vite; routing with React Router; state with Redux Toolkit (+ redux-persist), Zustand or TanStack Query depending on the project.
- Built complex UIs: 23 agent disposition panels, dynamic consultation forms driven by 16 JSON schemas with draft autosave, a 45-report BI dashboard with permission-filtered sidebar and virtualised tables, a WhatsApp chat UI with emoji picker/media/unread badges, cohort heatmap tables.
- Handled **iframe embedding** inside the Ameyo dialer (URL params + postMessage SSO, frame-ancestors headers, polling for session).
- Wrote axios clients with interceptors (JWT attach, 401 redirect, request-race protection, concurrency semaphore).
- Styled with Tailwind CSS (v3 and v4) and shadcn-style component primitives.

### Phase 6 — Testing & quality
- Manual + Postman testing (Postman collections committed), smoke-test scripts (`testConnections`, `runJobOnce`, `smoke_test`), health endpoints that report which env files loaded.
- Jest + Supertest set up for the CRM backend; ESLint + Prettier on all projects.
- Verified third-party API behaviour by live testing and documented real responses (e.g., an Ameyo API reference documenting that an auth failure returns a `null` body with HTTP 200).

### Phase 7 — Deployment & DevOps
- **Windows Server IIS + iisnode**: wrote `web.config` for each backend (handler, URL rewrite, process count, logging, watched files, max memory), configured app pools (No Managed Code, AlwaysRunning, idle timeout 0, preload), handled iisnode named-pipe PORT, ARR reverse proxy for `/api`, SPA rewrite rules and caching headers for the React build.
- **Cloudways (Linux)**: PM2 ecosystem config, Apache `.htaccess` reverse proxy including WebSocket upgrade, domain + SSL setup, deploy/check/test shell scripts.
- **Multi-environment configuration**: `.env.development / .env.staging / .env.production` for both frontend and backend; envalid validation to fail fast on missing config.
- Wrote **deployment runbooks** (IIS deploy doc, production checklist) so the setup is repeatable.

### Phase 8 — Production support & maintenance
- Diagnosed live issues from iisnode/Winston logs (example: config changes ignored after upload because iisnode did not watch `src` files → fixed `watchedFiles` and documented the app-pool restart command).
- Performance tuning after go-live (view bypass, index scripts, cache versioning, report range guard to stop the single Node process from freezing).
- Added features on request (consultation forms, courier companies, both live and archived recordings, phone masking).
- Kept a **"Never-500" error contract** for reporting apps: missing data degrades to `null` + a warning badge instead of crashing the page.

---

## 3. Role summary per project

| Project | My role | Team | Highlights of ownership |
|---|---|---|---|
| Appointment-CRM | Sole full-stack developer for the Node/React rewrite | Business users + legacy ASP.NET code as reference | Requirement parity analysis, 463-model code generation, 47 API modules, Ameyo/Exotel/SMS/courier integrations, IIS deployment |
| ViewReports | Sole full-stack developer / architect | Reporting stakeholders | 45+ reports, ETL sync engine, SQLite read-model, RBAC + OTP login, BullMQ worker, Excel exports, IIS deployment |
| WhatsApp Dashboard | Sole full-stack developer | Ops team | Exotel integration, webhooks, Socket.IO real-time, multi-number routing, reports, Cloudways deployment |
| Cohorts-Analytics | Sole developer (ported from Python) | Analytics owner | SQL translation MSSQL → SQLite, cohort math, styled Excel export, OTP 2FA, IIS + ARR |
| QMS | Sole developer | QA team | Two-leg call reconstruction, recording probe + streaming proxy, 92 s → 300 ms optimisation |
| Patient-CRM | Sole developer | Call-center agents | 3-tier cache, incremental SQLite ETL with PostgreSQL cursor, phone-variant matching |
| CallDropAutoDial | Sole developer | Call-center operations | SQL detection query, Ameyo batch upload, cron with DRY_RUN, IIS single-process deployment runbook |
| Email-Extract | Sole developer (MVP) | Self-initiated / product idea | BullMQ 4-stage pipeline, Puppeteer/Cheerio scraping, MX validation, JWT refresh tokens |

---

## 4. How to answer "Did you build all of this alone?"

> "Yes — the design, code, integrations and deployment were mine. I was not alone in the sense that I worked closely with business users for requirements and testing, and with the IT/network team for server access, dialer credentials and database permissions. But there was no other developer on these codebases; every commit is mine."

If asked about code review / team practices:
> "Because I was the only developer, I put extra effort into structure and documentation so someone else could take over: consistent layered folders, README/runbooks, OpenAPI docs, parity checklists, and comments that explain *why* a decision was made — for example why the IIS app runs only one Node process for the SQLite-backed apps."

---

## 5. How to answer "What was your daily work like?"

1. Morning: check production logs (iisnode / Winston / PM2), respond to issues from agents or managers.
2. Requirement discussion with the business owner of the feature (often 15–30 minutes, with screenshots of the legacy screen).
3. Backend first: SQL query → repository → service → controller → route + Joi validation → test with Postman.
4. Frontend: page/component → API service → state → UI polish.
5. Build (`vite build --mode production`), deploy to IIS or Cloudways, verify health endpoint, verify in the real Ameyo iframe / real WhatsApp number.
6. Commit with a clear message, update docs/checklists.

---

## 6. Skills you demonstrated beyond coding

- **Legacy system understanding** (reading C#/ASP.NET to extract business rules).
- **Data engineering basics** (ETL, read-models, cursors, keyset pagination, dedup logic).
- **Production mindset** (health checks, graceful shutdown, DRY_RUN, single-writer rules, error contracts, log-only policies decided with seniors).
- **Documentation** (runbooks, API references, checklists, specs).
- **Stakeholder communication** (turning vague requests like "QA needs to see everything about a call" into a 16-field report with defined rules).
- **Security awareness** (OTP 2FA, RBAC, phone masking, whitelisted proxy URLs, no open proxy, secrets in env files).
