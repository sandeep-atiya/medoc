# GitHub README Templates — one per project (safe for public view)

Before making any repo public: remove committed `.env*` files with secrets, internal IPs, database names, phone numbers and log files. Add `.env.example` only.

---

## Generic template

```markdown
# <Project Name>

<One-line description>

## Problem
<2–3 sentences on the business problem>

## Solution
<What the app does, bullet features>

## Architecture
<ASCII diagram or image>

## Tech Stack
- Frontend: ...
- Backend: ...
- Data: ...
- Deployment: ...

## Key Engineering Decisions
- ...

## Getting Started
```bash
cd backend && npm ci && cp .env.example .env && npm run dev
cd frontend && npm ci && npm run dev
```

## Environment Variables
| Name | Purpose |
|---|---|

## Deployment
<IIS / PM2 notes>

## Roadmap
- ...
```

---

## 1. Appointment-CRM

```markdown
# Appointment CRM — Call-Center CRM (React + Node + SQL Server)

Agent-facing CRM embedded in the Ameyo dialer for booking and managing clinic appointments, replacing a legacy ASP.NET WebForms application while keeping the existing SQL Server databases.

## Features
- Agent worklists: today, hold, follow-up, callback, non-contact, booking, feedback, search
- Appointments: create, reschedule, change clinic, audit trail
- Patients & call history, complaint tickets, IT issues, agent notes, attendance
- 16 disease-specific consultation forms (JSON-schema driven, draft autosave)
- WhatsApp (Exotel) + SMS confirmations with delivery-status webhook
- Courier tracking (Delhivery, XpressBees, ShipDelight)
- Statistics dashboard, RBAC, OpenAPI docs at /api/docs, health/readiness endpoints

## Tech
React 19 · Vite · Redux Toolkit · Tailwind · Recharts | Node · Express 5 · Joi · Winston · Swagger | SQL Server (mssql) ×2 pools · Redis | IIS + iisnode / PM2

## Highlights
- `scripts/generate-models-from-sql.js`: DDL → 463 model classes
- Layered: routes → validation → controller → service → repository
- Stateless iframe SSO from the dialer; retry/backoff for WhatsApp
```

## 2. ViewReports

```markdown
# ViewReports — Call-Center BI Platform

45+ reports over dialer/ACD and order data. Chunked incremental ETL from PostgreSQL and SQL Server into a SQLite read-model; JWT + email OTP; per-report RBAC; Excel/CSV export.

## Architecture
SQL Server + PostgreSQL → sync engine (cron / BullMQ worker) → SQLite (WAL) → Express API (range guard, cache) → React SPA

## Tech
React 19 · TanStack Query/Table/Virtual · Zustand · Tailwind 4 · Recharts | Express 5 · Joi · envalid · Winston · BullMQ · ExcelJS | SQL Server · PostgreSQL · SQLite · Redis | IIS + iisnode

## Notable
- Worker-thread partial index build; single-writer design
- Cache keyed by last_sync_at; export guards (800k/1M rows)
- Admin sync console: progress, cancel, crash recovery
```

## 3. WhatsApp Dashboard

```markdown
# WhatsApp Dashboard — Real-Time Business Inbox (MERN + Socket.IO)

Shared inbox for WhatsApp Business numbers via the Exotel WhatsApp API: webhooks → MongoDB → Socket.IO → agents. Automatic multi-number routing, unread tracking, Cloudinary media, aggregation reports.

## Tech
React 19 · Vite · Tailwind · socket.io-client | Express 4 · Socket.IO · Mongoose 7 · Joi · multer · Cloudinary | MongoDB Atlas | Cloudways · PM2 · Apache (.htaccess proxy + WebSocket)

## Run locally
- `backend`: `npm i && npm run dev` (needs Mongo URI, Exotel keys, Cloudinary URL)
- `frontend`: `npm i && npm run dev`
- Expose webhook with `ngrok http 5000`
```

## 4. Cohorts-Analytics

```markdown
# Cohorts Analytics — Retention & LTV Dashboard

Mirrors order data from SQL Server into SQLite (keyset pagination), computes monthly cohort retention (NTB & overall modes), KPIs and styled Excel exports. JWT + email OTP login.

## Tech
React 19 · Vite | Express 5 · better-sqlite3 · mssql · node-cron · nodemailer · ExcelJS | IIS + iisnode + ARR
```

## 5. QMS

```markdown
# QMS — Call-Quality Audit Report

Enter a Call ID → complete audit sheet: both call legs, agents, durations, hangup, disposition, patient + clinical capture, NTB/Repeat, and an in-page call recording streamed through a Range-aware proxy.

## Tech
React 19 · TanStack Query · Tailwind 4 | Express 5 · envalid · Winston | PostgreSQL · SQL Server · Redis | Ameyo voice-log & archive APIs | IIS + iisnode

## Notable
- Two-leg reconstruction via interaction id
- Recording probe (ranged GET, tri-state availability)
- 92 s → 300 ms via view bypass + indexes (see `sql/`)
```

## 6. Patient-CRM

```markdown
# Patient CRM — Call-History Lookup

Search a phone number or patient ID → 30-day call history across all the patient's numbers. Three-tier cache: LRU → Redis → SQLite index (built from PostgreSQL with server-side cursors) → PostgreSQL.

## Tech
React 19 · Tailwind | Express 5 · Joi · lru-cache · ioredis · mssql · pg · sqlite3 | IIS + iisnode
```

## 7. CallDropAutoDial

```markdown
# CallDropAutoDial — Dropped-Call Recovery Job

Every 30 minutes (business hours) selects dropped/abandoned inbound calls from the Ameyo ACD table (PostgreSQL), de-duplicates, and uploads them to the right Ameyo outbound campaign via `uploadContacts`. DRY_RUN by default.

## Endpoints
- `GET /api/health` · `GET /api/call-drops?minutes=N` (preview) · `POST /api/job/run` (manual)

## Tech
Node 18+ · Express 5 · pg · node-cron · envalid · Winston | IIS + iisnode (single process)
```

## 8. Email-Extract

```markdown
# Email Extract — Lead-Generation Pipeline (Prototype)

Create a project → enter a search query → BullMQ pipeline (SerpAPI → scrape with Cheerio/Puppeteer → validate emails via regex, disposable list, DNS MX → store). JWT access/refresh auth, Redux Toolkit + React Query dashboard.

## Tech
React 19 · Redux Toolkit · React Query · react-hook-form · Tailwind 4 · Recharts | Express 4 · Mongoose 8 · BullMQ 5 · ioredis · Puppeteer · Cheerio · Winston | MongoDB · Redis

## Status
MVP — export UI and a few fixes pending (see Issues).
```
