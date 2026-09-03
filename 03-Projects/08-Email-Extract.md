# Project 8 — Email-Extract (Lead-Generation SaaS: Search → Scrape → Validate, BullMQ pipeline)

**Repo:** `D:\email-extract\email-extract` (nested folder) — 1 commit ("Initial commit"), 105 files
**Period:** June 2026 (MVP / product prototype)
**Size:** ~93 source files, ~3,400 lines
**My role:** Sole developer — product idea, MongoDB schema, 4-stage BullMQ pipeline, scraping engine (Cheerio + Puppeteer), email validation ladder, JWT access/refresh auth, React dashboard.

> Present this as a **self-built SaaS prototype / MVP**, not a production system. It shows queue design, scraping, and MongoDB skills.

---

## 1. Pitches

**One line:**
> A lead-generation platform: a user creates a project, enters a Google search query, and a BullMQ pipeline fetches result URLs via SerpAPI, scrapes each site for email addresses (Cheerio first, Puppeteer fallback), validates them (format → disposable domain → MX record), de-duplicates and stores them per project with a dashboard.

**30-second pitch:**
> "Email-Extract is an MVP I built to learn and demonstrate queue-based architecture. The API is Express 4 with MongoDB/Mongoose and Redis; scraping work runs in four BullMQ queues — SERP, scraper, email, export — each with its own retry and backoff policy and worker concurrency. The scraper tries fast HTML parsing with Cheerio, probes contact/about pages, and only falls back to headless Puppeteer when needed. Emails are validated with a regex, a disposable-domain list and DNS MX lookup, then deduped at three levels including a unique compound index. Auth is JWT with short-lived access tokens and refresh tokens. The React frontend uses Redux Toolkit, React Query, react-hook-form and Tailwind 4 with a Recharts dashboard."

---

## 2. Problem / idea

- Sales teams need lists of business emails for a niche (e.g., "marketing agencies in New York").
- Manual searching and copy-paste is slow and error-prone; results contain fake or disposable addresses.
- Goal: a metered multi-user tool (users have email credits and roles).

## 3. What I built — features

- **Auth:** register/login (rate-limited, Joi), refresh token (7 d) + access token (15 min) with separate secrets, logout, `me`; bcrypt with 12 rounds; refresh token stored on the user.
- **Projects:** CRUD scoped by owner (every query filters by `owner: userId` → no IDOR).
- **Scraping jobs:** start (rate-limited 30/hour), list, get, cancel; job status pending/running/completed/failed/cancelled with timing.
- **Pipeline:** `POST /scraping/start` → create Search → **serpQueue** (SerpAPI, paginated by 10) → fan-out one job per URL → **scraperQueue** (concurrency 5, 3 attempts, exponential 5 s) → **emailQueue** (concurrency 3, batch validate + `insertMany` unordered) → project counters updated.
- **Scraper fallback chain:** axios + Cheerio with rotating user agents and 15 s timeout → probe `/contact`, `/contact-us`, `/about`, `/team`, `/about-us` → Puppeteer headless (`networkidle2`, 30 s) if still empty.
- **Validation ladder:** regex → strict anchored check → lowercase set → disposable-domain set → `dns.resolveMx` → status `valid / invalid / risky / disposable / unverified`.
- **Dedup:** in-memory Set → DB `$in` check against existing project emails → unique compound index `{address, project}` → `insertMany({ordered:false})` so one duplicate does not abort the batch.
- **Emails API:** list (paginated), stats by status (aggregation), get one, delete by project.
- **Dashboard:** 4 stats cards, 7-day jobs area chart (`$dateToString` aggregation), projects page, scraper page with live job list (poll 5 s) and cancel, results page with status tiles and paginated table, settings page.
- **Ops:** Winston logging with exception/rejection handlers, centralised error mapping (Mongoose validation → 422, cast → 400, duplicate key → 409, JWT errors → 401), 3 rate-limit tiers, Redis version guard (if Redis < 5, API starts with queues disabled instead of crashing), graceful shutdown.

## 4. Architecture

```
React 19 SPA (Redux Toolkit auth/ui + React Query server state, react-hook-form, Tailwind 4, Recharts)
        ▼ axios (Bearer access token; 401 → clear + redirect)
Express 4 API (/api/v1): auth · projects · scraping · projects/:id/emails · dashboard
   middlewares: helmet, hpp, cors, compression, express-rate-limit (global/auth/scrape), Joi validate, protect (JWT)
        ▼                                    ▼
MongoDB (Mongoose 8): User, Project,      Redis (ioredis) ── BullMQ queues: serp → scraper → email (→ export)
Search, ScrapeJob, Email, Export           workers: serp(2) · scraper(5) · email(3) · export(2)
                                                       │
                              SerpAPI (Google results) · axios+Cheerio · Puppeteer · DNS MX
```

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 8 + React Compiler, react-router-dom 7, Redux Toolkit 2 + react-redux, TanStack React Query 5, react-hook-form 7, Tailwind CSS 4, Recharts 3, axios |
| Backend | Node.js ESM, Express 4.18, Mongoose 8, BullMQ 5 + ioredis, Puppeteer 21, Cheerio, axios, user-agents, google-search-results-nodejs (SerpAPI), jsonwebtoken, bcryptjs, Joi 17, Winston + morgan, helmet, hpp, cors, compression, express-rate-limit, ExcelJS, csv-writer, pdfkit, `dns/promises` |
| Data | MongoDB (6 collections with compound indexes), Redis (queue broker) |
| Deploy | Local / MVP — `.env.example` for both apps; no production deployment yet |

## 6. Database design (Mongoose)

| Model | Key fields | Indexes |
|---|---|---|
| User | name, email (unique), password (select:false), role admin/user, isVerified, isActive, refreshToken (select:false), lastLogin, emailCredits (100), totalScrapes | unique email |
| Project | name, description, owner→User, totalEmails, totalSearches, isArchived, tags[] | {owner, createdAt:-1} |
| Search | query, project, user, engine, totalResults, urlsFound, emailsFound, urls[], serpData | {project, createdAt:-1}, {user} |
| ScrapeJob | user, project, search, targetUrl, status, emailsFound, errorMessage, startedAt, completedAt, duration, bullJobId | {user,status}, {project, createdAt:-1} |
| Email | address, domain, project, user, scrapeJob, sourceUrl, status, isDisposable, mxValid, firstName, lastName, company, jobTitle | unique {address, project}, {project,status}, {domain} |
| Export | user, project, format csv/excel/pdf, status, filePath, fileName, totalRecords | {user, createdAt:-1}, {project} |

## 7. Challenges and solutions

1. **Scraping is slow and flaky.** → Queue with per-URL jobs, concurrency 5, retries with exponential backoff; Cheerio-first for speed, Puppeteer only as fallback.
2. **Duplicate emails across pages of the same site.** → Three dedup layers + unique index + unordered insertMany.
3. **Fake/disposable emails pollute lists.** → Validation ladder with MX lookup; `risky` status when DNS fails rather than dropping.
4. **Redis on the dev machine was old.** → Version guard: API boots with queues disabled and logs the fix, instead of crashing.
5. **Users must not see each other's data.** → Every query scoped by owner/user.

## 8. Interview questions specific to this project

**Q: Why BullMQ instead of running scraping in the request?**
A: Scraping takes seconds to minutes per site; the request would time out and the server would block. Queues give retries, concurrency control, progress and cancellation.

**Q: How do you avoid getting blocked by target sites?**
A: Rotating user agents, timeouts, limited concurrency, Cheerio before a heavy browser, and respecting failures with backoff. For production I would add proxies and robots.txt checks.

**Q: Why both access and refresh tokens?**
A: Short access token limits damage if leaked; refresh token lets the UI stay logged in and can be revoked by clearing it on the user document.

**Q: How would you deploy this?**
A: API + workers as separate PM2 processes (or containers), MongoDB Atlas, managed Redis, Puppeteer with a Chromium image; frontend as static files behind Nginx/Apache — same pattern as my Cloudways deployment.

**Q: How do you cancel a running job?**
A: Currently the Mongo status is set to cancelled for pending jobs; the improvement is to also remove/abort the BullMQ job by `bullJobId`.

## 9. Be careful — what NOT to claim

- It is an **MVP** — one commit, not deployed. Say "prototype I built to demonstrate queue architecture".
- **Export** (CSV/Excel/PDF) models, queue, worker and services exist but there is **no export route/button** wired yet. Say "export pipeline scaffolded, UI pending".
- nodemailer and node-cron are installed but unused — no email sending, no scheduler.
- Known bugs you should be aware of if asked to demo: aggregation `$match` uses string ids (needs `ObjectId` cast) so some stats return zeros; `useAuth` uses an `onSuccess` option removed in React Query v5; job cancel does not remove the BullMQ job. Presenting these as "known issues in my backlog" is honest and fine.

## 10. What I would improve next

- Fix the `ObjectId` cast in aggregations and the React Query v5 hook; wire the export route + button.
- Proxy rotation, robots.txt compliance, per-domain rate limiting.
- Credits deduction per validated email; Stripe billing; Docker compose for API + workers + Redis + Mongo.

## 11. Resume bullets

- Built a lead-generation SaaS prototype (React 19 + Express 4 + MongoDB + Redis) with a four-stage BullMQ pipeline (SerpAPI search → Cheerio/Puppeteer scraping → email validation with disposable-domain and DNS MX checks → storage) using per-queue retry/backoff, worker concurrency and multi-layer de-duplication.
- Implemented JWT access/refresh authentication, owner-scoped data access, tiered rate limiting, Winston logging with centralised error mapping, and a Redux Toolkit + React Query dashboard with Recharts.

## 12. Keywords

BullMQ, Redis, job queues, workers, web scraping, Puppeteer, Cheerio, SerpAPI, email validation, DNS MX, MongoDB, Mongoose, compound indexes, aggregation, JWT refresh tokens, bcrypt, rate limiting, React Query, Redux Toolkit, react-hook-form, Tailwind 4, Recharts, SaaS
