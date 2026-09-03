# Master Overview — All 8 Projects

Use this file to remember the big picture. Every number here comes from the actual code.

---

## 1. The business context (say this in every interview)

All 8 applications were built for **a healthcare tele-consultation company** that runs:

- A **large call center** using the **Ameyo dialer** (inbound + outbound campaigns, agents, queues, dispositions).
- **Physical clinics** where patients book appointments through call-center agents.
- **E-commerce style medicine orders** with courier tracking (Delhivery, XpressBees, ShipDelight, Shiprocket).
- **WhatsApp and SMS** customer communication through **Exotel** (WhatsApp Business API) and **ValueFirst** (SMS).

The company had an **old ASP.NET WebForms + SQL Server** system. My job was to **modernise it piece by piece using Node.js + React**, keep parity with the legacy behaviour, and add new analytics/automation tools on top of the existing databases (SQL Server for CRM data, PostgreSQL for the dialer data warehouse).

One sentence version:
> "I modernised a healthcare call-center's legacy ASP.NET platform into a set of Node.js + React applications — CRM, reporting/BI, quality monitoring, WhatsApp inbox and automation jobs — and deployed all of them myself on Windows IIS and Cloudways."

---

## 2. Project summary table

| # | Project | Type | Frontend | Backend | Databases / Cache | Key integrations | Deploy | Size (approx) |
|---|---|---|---|---|---|---|---|---|
| 1 | Appointment-CRM | Agent CRM (replaces legacy ASP.NET) | React 19, Vite, Redux Toolkit + redux-persist, React Router 6, Tailwind 3, Recharts | Node, Express 5 (ESM), Joi, Winston, Swagger/OpenAPI | SQL Server ×2 (dual pool), Redis Cloud | Ameyo dialer API + iframe SSO, Exotel WhatsApp, ValueFirst SMS, courier tracking APIs | IIS + iisnode (4 node processes), PM2 config also present | ~934 files, ~78k lines |
| 2 | ViewReports | BI / reporting platform | React 19, Vite 8, TanStack Query/Table/Virtual, Zustand, Tailwind 4, shadcn-style UI, Recharts, react-hook-form + zod | Express 5, Joi, envalid, Winston, BullMQ worker, node-cron | SQL Server + PostgreSQL + SQLite (better-sqlite3 read-model) + Redis | Ameyo voice-log URLs, Gmail SMTP (OTP) | IIS + iisnode (single process, deliberate) | ~435 files, ~59k lines |
| 3 | WhatsApp Dashboard | Real-time chat inbox | React 19, Vite 7, Tailwind 3, socket.io-client, emoji picker, dropzone | Express 4, Socket.IO 4, Mongoose 7, Joi, multer | MongoDB Atlas | Exotel WhatsApp Business API v2 + webhooks, Cloudinary media | Cloudways (Linux) PM2 + Apache .htaccess proxy | ~65 files, ~7k lines |
| 4 | Cohorts-Analytics | Retention / LTV analytics | React 19, Vite 8, Context + useReducer | Express 5, node-cron, nodemailer, exceljs | SQL Server (source) → SQLite mirror (better-sqlite3) | Gmail SMTP OTP | IIS + iisnode + ARR reverse proxy | ~24 files, ~2.7k lines |
| 5 | QMS | Call-quality audit report | React 19, Vite 8, TanStack Query, Tailwind 4 | Express 5, envalid, Winston | PostgreSQL + SQL Server + Redis (versioned cache key) | Ameyo live voice-log + archive APIs (audio streaming proxy) | IIS + iisnode | ~65 files, ~2.4k lines |
| 6 | Patient-CRM | Call-history lookup | React 19, Vite 8, Tailwind 3 | Express 5, Joi | PostgreSQL + SQL Server + SQLite + Redis + in-memory LRU | (DB level only) | IIS + iisnode | ~71 files, ~2.5k lines |
| 7 | CallDropAutoDial | Scheduled automation job | none (API only) | Express 5, node-cron, envalid, Winston | PostgreSQL | Ameyo uploadContacts API | IIS + iisnode (1 process, so cron never double-fires) | ~35 files, ~2.3k lines |
| 8 | Email-Extract | Lead-gen SaaS (MVP) | React 19, Vite 8, Redux Toolkit, React Query, react-hook-form, Tailwind 4, Recharts | Express 4, Mongoose 8, BullMQ 5, Puppeteer, Cheerio, Winston | MongoDB + Redis | SerpAPI (Google results), DNS MX lookups | Not deployed yet | ~93 files, ~3.4k lines |

**Total: roughly 1,700 source files and ~155,000 lines of code across 8 repositories.**

---

## 3. Tech-stack matrix (what you can honestly put on your resume)

### Frontend
| Skill | Used in |
|---|---|
| React 19 (hooks, Context, lazy routes, Suspense) | All 7 UI projects |
| Vite (7/8), React Compiler (babel-plugin-react-compiler) | ViewReports, QMS, Patient-CRM, Cohorts, Email-Extract |
| Redux Toolkit + redux-persist | Appointment-CRM, Email-Extract |
| Zustand | ViewReports |
| TanStack React Query v5 | ViewReports, QMS, Email-Extract |
| TanStack Table + Virtual (virtualised big tables) | ViewReports |
| React Router 6/7 | All UI projects |
| Tailwind CSS 3 and 4, shadcn-style components (cva, clsx, tailwind-merge) | All UI projects |
| Recharts | Appointment-CRM, ViewReports, Email-Extract |
| react-hook-form + zod | ViewReports, Email-Extract |
| Socket.IO client (real-time) | WhatsApp Dashboard |
| Axios interceptors (JWT, 401 handling, API-key header) | All |
| Iframe embedding / postMessage SSO | Appointment-CRM, Patient-CRM, QMS |

### Backend
| Skill | Used in |
|---|---|
| Node.js (ESM), Express 5 | Appointment-CRM, ViewReports, QMS, Patient-CRM, Cohorts, CallDropAutoDial |
| Express 4 | WhatsApp Dashboard, Email-Extract |
| Layered architecture: routes → validation → controller → service → repository | Appointment-CRM (47 modules), ViewReports (48 modules), all others |
| Joi validation, envalid (fail-fast env config) | Most projects |
| JWT auth + bcrypt | Appointment-CRM, ViewReports, Cohorts, Email-Extract |
| Email OTP two-factor login | ViewReports, Cohorts-Analytics |
| Role/permission based access (RBAC) | Appointment-CRM, ViewReports |
| Socket.IO server | WhatsApp Dashboard |
| BullMQ queues + workers | Email-Extract (4-stage pipeline), ViewReports (sync worker) |
| node-cron scheduled jobs | CallDropAutoDial, Cohorts, ViewReports |
| Winston logging (daily rotate), Morgan | Most projects |
| Helmet, CORS, rate limiting, hpp, compression | Most projects |
| Swagger / OpenAPI docs | Appointment-CRM |
| Retry with exponential backoff | Appointment-CRM (WhatsApp), WhatsApp Dashboard (Exotel) |
| Graceful shutdown, health/readiness endpoints | Most projects |
| Web scraping: Puppeteer + Cheerio | Email-Extract |
| Excel export with ExcelJS (styled heatmaps, 800k row guard) | ViewReports, Cohorts |
| Audio streaming proxy with HTTP Range support | QMS |
| Code generation scripts (SQL DDL → models → controllers) | Appointment-CRM |

### Databases & caching
| Skill | Used in |
|---|---|
| Microsoft SQL Server (mssql/tedious, raw parameterised SQL, dual connection pools) | Appointment-CRM, ViewReports, QMS, Patient-CRM, Cohorts |
| PostgreSQL (pg, server-side cursors, DISTINCT ON, NOT EXISTS, partial indexes) | ViewReports, QMS, Patient-CRM, CallDropAutoDial |
| MongoDB + Mongoose (schemas, compound indexes, aggregation pipelines) | WhatsApp Dashboard, Email-Extract |
| SQLite (better-sqlite3, WAL mode) as local read-model / cache | ViewReports, Cohorts, Patient-CRM |
| Redis (ioredis) for caching, OTP, sessions, BullMQ broker | ViewReports, QMS, Patient-CRM, Appointment-CRM, Email-Extract |
| ETL / data sync (keyset pagination, chunked incremental sync, cursors) | ViewReports, Cohorts, Patient-CRM |
| Query optimisation (index design, view bypass: 92 s → 300 ms) | QMS, ViewReports |
| Dynamic data masking awareness (SQL Server UNMASK) | Patient-CRM, QMS |

### Deployment / DevOps
| Skill | Used in |
|---|---|
| Windows Server IIS + iisnode (web.config, app pools, AlwaysRunning, named pipes, URL Rewrite) | 6 projects |
| IIS ARR reverse proxy (`/api` → Node) | Cohorts-Analytics |
| SPA hosting on IIS (rewrite to index.html, caching headers) | All IIS projects |
| Cloudways (Linux) + PM2 + Apache .htaccess reverse proxy + WebSocket proxy | WhatsApp Dashboard |
| PM2 ecosystem files (cluster and fork modes) | Appointment-CRM, WhatsApp Dashboard |
| Multi-environment env files (.env.development / staging / production) | All |
| Git + GitHub | All |
| ngrok for webhook testing | WhatsApp Dashboard |
| Postman collections, smoke-test scripts | WhatsApp Dashboard, QMS, CallDropAutoDial |

### Third-party integrations
Ameyo dialer (REST commands, uploadContacts, voice-log/archiver, iframe SSO) · Exotel WhatsApp Business API (messages, templates, DLR webhooks) · ValueFirst SMS gateway · Cloudinary · SerpAPI · Courier APIs (Delhivery, XpressBees, ShipDelight, Shiprocket) · Gmail SMTP · MongoDB Atlas · Redis Cloud

---

## 4. Timeline (from git history and file dates)

| Period | What happened |
|---|---|
| Nov–Dec 2025 | Started **WhatsApp Dashboard** (first Node + React + Socket.IO app, deployed on Cloudways). Started **Appointment-CRM** rewrite. |
| Jan–Feb 2026 | Heavy development on Appointment-CRM (100+ commits in Jan). WhatsApp Dashboard multi-number routing + reports done. |
| Mar 2026 | Appointment-CRM stabilisation on IIS. |
| May 2026 | Started **Patient-CRM** (call-history lookup with 3-tier cache). |
| Jun 2026 | **Cohorts-Analytics** (ported Python analytics to Node), **Email-Extract** MVP, started **ViewReports**. |
| Jul–Aug 2026 | ViewReports main build (75 commits), **QMS** call-quality report, **CallDropAutoDial** automation deployed live. |
| Sep 2026 | Appointment-CRM consultation forms + WhatsApp fixes live; QMS recording streaming fixes live. |

---

## 5. Common architecture pattern (explain this once, apply to all)

```
Browser (React SPA, Vite build, served by IIS/Apache as static files)
        │  HTTPS/HTTP  (axios, JWT / API key / iframe SSO headers)
        ▼
Node.js + Express API
   routes  →  validation (Joi)  →  controller  →  service (business logic)  →  repository (SQL / Mongoose)
   middlewares: helmet, cors, rate-limit, auth, error handler, request logging (Winston/Morgan)
        │
        ├── SQL Server (CRM: patients, appointments, orders, users, permissions)
        ├── PostgreSQL (Ameyo dialer / ACD call warehouse, 50M+ rows)
        ├── SQLite (local read-model cache for heavy reports)
        ├── Redis (cache, OTP, sessions, BullMQ)
        └── MongoDB (chat messages, scraped leads)
        │
        └── External APIs: Ameyo, Exotel WhatsApp, ValueFirst SMS, Cloudinary, SerpAPI, courier APIs
```

**Why this pattern?** Separation of concerns; every report/feature is 5 small files instead of one big file; easy to test; easy for a new developer to find code; the same pattern was reused across 8 projects which made me faster on each new project.

---

## 6. The five "hero stories" to always have ready

1. **Legacy migration with parity** (Appointment-CRM): 463 SQL tables → auto-generated models via a script I wrote; a parity checklist against the ASP.NET code; agents kept working inside the Ameyo iframe with zero training change.
2. **Reporting at scale** (ViewReports): 50M+ row tables; built a chunked ETL into a local SQLite read-model; partial index built in a worker thread; report date-range guard; 800k-row Excel export guard; in-process cache invalidated by the sync timestamp.
3. **Query optimisation** (QMS): SQL Server view with per-row OUTER APPLY took ~92 seconds; I bypassed the view, queried base tables with UNION ALL and added indexes → 200–350 ms.
4. **Real-time messaging** (WhatsApp Dashboard): Exotel webhooks → MongoDB → Socket.IO broadcast; automatic multi-business-number routing so replies go out from the same number the customer wrote to; retry with exponential backoff.
5. **Safe automation** (CallDropAutoDial): cron every 30 min; DRY_RUN flag; re-entrancy guard; single IIS worker process to avoid duplicate pushes; found and fixed a data-source problem (the child table was filled by a 1-hour ETL so the 30-minute window was always empty → switched to the live parent table).
