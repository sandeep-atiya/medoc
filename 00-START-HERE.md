# START HERE — Your Interview, Resume, Portfolio & LinkedIn Kit

This folder was built by reading the actual source code of your 8 applications on the D: drive.
Everything written here is based on what really exists in the code, so you can say it with confidence in an interview.

---

## 1. What is inside this folder

| Folder / File | What it is for | When to read it |
|---|---|---|
| `00-START-HERE.md` | This file. Index + how to prepare | First |
| `01-Master-Overview-All-Projects.md` | One-page summary of all 8 projects, tech stack matrix, timeline | Before any interview |
| `02-My-Roles-and-Responsibilities.md` | Your end-to-end role: requirement → design → code → deploy → support | Before any interview |
| `03-Projects/` | One detailed file per project (architecture, features, challenges, Q&A, pitch) | Deep preparation |
| `04-Interview-QA/` | Question banks with answers: React, Node, DB, deployment, security, system design, behavioral, HR, tricky questions | Daily revision |
| `05-Resume-Portfolio/` | Ready resume bullets, skills section, portfolio text, GitHub README templates | When updating resume/portfolio |
| `06-LinkedIn/` | 10 ready LinkedIn posts, headline, About section, hashtags, posting plan | When posting |
| `07-Deployment-Playbook-IIS-and-Cloudways.md` | Step-by-step how you deploy on IIS (iisnode) and Cloudways (PM2 + Apache) | Before DevOps questions |
| `08-Quick-Revision-Cheat-Sheet.md` | 2-page revision before walking into the interview | 30 minutes before interview |
| `09-Glossary.md` | Simple meaning of every technical word used in these docs | Whenever a word is unclear |

---

## 2. Your 8 applications at a glance

| # | Project | One line | Main stack | Deployed on |
|---|---|---|---|---|
| 1 | **Appointment-CRM** | Call-center CRM for clinic appointments, replacing a legacy ASP.NET app. Agents work inside the Ameyo dialer. | React 19 + Redux Toolkit + Tailwind / Node + Express 5 + SQL Server (2 DBs) + Redis | Windows Server, IIS + iisnode |
| 2 | **ViewReports** | Enterprise call-center BI platform with 45+ reports, ETL sync, Excel export, RBAC, OTP login | React 19 + TanStack + Zustand / Express 5 + SQL Server + PostgreSQL + SQLite cache + Redis + BullMQ | IIS + iisnode |
| 3 | **WhatsApp Dashboard** | Real-time WhatsApp Business inbox for agents (Exotel API), multi-number routing, reports | React 19 + Socket.IO client / Express 4 + MongoDB Atlas + Socket.IO + Cloudinary | Cloudways (Linux), PM2 + Apache |
| 4 | **Cohorts-Analytics** | Customer cohort retention & LTV dashboard, ported from Python, Excel heatmap export, OTP 2FA | React 19 / Express 5 + SQL Server → SQLite mirror + node-cron | IIS + iisnode + ARR reverse proxy |
| 5 | **QMS (Quality Monitoring)** | Call-quality audit report per Call ID with two-leg call reconstruction and in-page call recording playback | React 19 + TanStack Query + Tailwind 4 / Express 5 + PostgreSQL + SQL Server + Redis + Ameyo APIs | IIS + iisnode |
| 6 | **Patient-CRM** | Agent lookup: search a phone/patient ID → full 30-day call history across all patient numbers, 3-tier cache | React 19 / Express 5 + PostgreSQL + SQL Server + SQLite + Redis (LRU → Redis → SQLite → PG) | IIS + iisnode |
| 7 | **CallDropAutoDial** | Backend job: every 30 min finds dropped/abandoned calls in PostgreSQL and pushes them to Ameyo campaigns for auto call-back | Node + Express 5 + PostgreSQL + node-cron + Winston | IIS + iisnode |
| 8 | **Email-Extract** | Lead-generation SaaS: Google search (SerpAPI) → scrape sites → extract & validate emails via BullMQ pipeline | React 19 + Redux Toolkit + React Query / Express 4 + MongoDB + Redis + BullMQ + Puppeteer + Cheerio | Local / MVP (not yet deployed) |

**Timeline:** December 2025 → September 2026 (about 9–10 months, all built by you end-to-end).

---

## 3. How to use this kit (7-day plan)

| Day | Do this |
|---|---|
| 1 | Read `01-Master-Overview` and `02-Roles-and-Responsibilities`. Memorise the one-line description of every project. |
| 2 | Read `03-Projects/01-Appointment-CRM.md` and `03-Projects/02-ViewReports.md` (your two biggest projects). Practise the 30-second and 2-minute pitch out loud. |
| 3 | Read the remaining 6 project files. For each, remember: problem → solution → stack → one challenge → one result. |
| 4 | `04-Interview-QA/03-React` + `04-Node-Express` + `05-Database`. Answer out loud without reading. |
| 5 | `04-Interview-QA/06-Deployment` + `07-Deployment-Playbook`. Be able to explain IIS + iisnode and Cloudways + PM2 from memory. |
| 6 | `04-Interview-QA/07-Security`, `08-System-Design`, `10-Tricky-Questions`. These are where interviewers try to catch you. |
| 7 | `09-Behavioral-STAR`, `11-HR-Round`, then `08-Quick-Revision-Cheat-Sheet`. Update resume using `05-Resume-Portfolio`. |

---

## 4. Golden rules for the interview (very important)

1. **Only claim what really exists.** Each project file has a section called *"Be careful — what NOT to claim"*. Interviewers often ask follow-up questions; if you claim Socket.IO in a project that only has it installed but unused, you will get caught.
2. **Always answer in this order:** Business problem → What I built → Tech stack → One hard challenge and how I solved it → Result/impact.
3. **Use numbers.** 45+ reports, 47 API modules, 463 auto-generated models, 50M+ row tables, 92 seconds → 300 ms query, 800,000-row Excel export guard. All are real (see project files).
4. **Own the full lifecycle.** You gathered requirements from business users, designed the DB/API, built frontend + backend, deployed to IIS/Cloudways, and supported it in production. Say this clearly.
5. **Turn weaknesses into improvements.** If asked "why no authentication in QMS?", answer: "It runs inside an authenticated host app on the LAN; the next step in my roadmap is JWT middleware shared with ViewReports." Every project file has "What I would improve next".
6. **Do not share confidential data.** Do not put internal IPs, database names, passwords, or client business numbers on LinkedIn or a public portfolio. The LinkedIn posts here are already written in a safe, generic way.

---

## 5. Before you publish anything — two things to check

1. **Name and GitHub account.** The git history of these repositories shows commits under the author name *Sandeep* and the GitHub account `sandeep-atiya`. If an interviewer or recruiter opens the GitHub links, they will see that. Make sure your resume/portfolio links point to a GitHub account that is yours, or be ready to explain that the code lives under a company/team account. All documents here use the placeholder **[Your Name]** — do a Find & Replace.
2. **Client name.** The applications are built for a healthcare / Ayurvedic tele-consultation business (clinics + call center + e-commerce orders). The docs use the safe phrase *"a healthcare tele-consultation company"*. Only mention the real client name if your company allows it.

---

## 6. Other folders on D: not covered

`ameyo_report`, `CallDropAudioDial` (older copy), `ConsultationForm`, `QMS-Recording-API`, `UnayurIN_NEW`, `Bk` exist on the drive but were not in your list, so they are not documented here. If any of them is a separate deliverable you want to present, tell me and I will add a project file for it.
