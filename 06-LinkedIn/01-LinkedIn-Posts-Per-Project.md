# LinkedIn Posts — 10 ready-to-publish posts

Rules used: no client names, no internal hosts/IPs, no secrets. Hook in the first line (LinkedIn shows ~2 lines before "see more"). Short paragraphs. End with a question. 3–6 hashtags.
Post 1–2 per week. Attach a diagram or blurred screenshot if you can.

---

## Post 0 — Series intro

Over the last 9 months I built and deployed 8 production applications — alone, end-to-end.

CRM. BI reporting. Real-time WhatsApp inbox. Analytics. Automation.
All for one healthcare call-center business, on top of its existing SQL Server and PostgreSQL data.

Stack: React 19 · Node.js/Express · SQL Server · PostgreSQL · MongoDB · SQLite · Redis · Socket.IO · BullMQ · IIS · PM2.

I'm going to share one project per post — the problem, the design, the hard part, and what I'd do differently.

Post 1 tomorrow: replacing a legacy ASP.NET WebForms CRM without touching the database.

Which one sounds most interesting to you? 👇

#FullStackDeveloper #NodeJS #ReactJS #SoftwareEngineering #BuildInPublic

---

## Post 1 — Appointment-CRM (legacy migration)

I replaced a 250 KB C# code-behind file with a React + Node app — and agents didn't notice the switch.

The situation: call-center agents booked clinic appointments in an ASP.NET WebForms page that opened inside the dialer. Slow, fragile, hard to change. But 460+ tables across two SQL Server databases were shared with other systems, so "rewrite the database" was not an option.

What I did:
→ Kept the databases. Built a new Express 5 API + React 19 SPA on top.
→ Wrote a script that reads the SQL DDL and generates a model class per table — 463 models, zero hand-typed column names.
→ Mirrored the legacy behaviour with a parity checklist (even the SMS gateway still returns "Y"/"N").
→ Zero-click login: the dialer passes the agent session into the iframe; a middleware resolves it server-side.
→ Appointment confirmations on WhatsApp (Exotel) + SMS, with retries only on transient errors and delivery-status webhooks.

Result: 47 API modules, 23 agent panels, 16 dynamic consultation forms, OpenAPI docs, deployed on IIS with iisnode.

Biggest lesson: in a migration, parity is the feature. New tech is invisible to the user — and that's the goal.

Have you migrated a legacy app while keeping the database? What was your hardest parity bug?

#LegacyModernization #NodeJS #ReactJS #SQLServer #SoftwareArchitecture

---

## Post 2 — ViewReports (reporting at scale)

Reports on a 50-million-row table used to take minutes. Now they take milliseconds. Here's the design.

Problem: management needed 45+ reports across the dialer warehouse (PostgreSQL) and the CRM (SQL Server). Running heavy SQL against production was slow and risky.

Design:
1. A sync engine pulls data in chunks (cursor per source) into a local SQLite read-model with the right indexes.
2. Every report is 5 small files: route → validation → controller → service → repository.
3. Cache key includes the last-sync timestamp — invalidation is automatic after every sync.
4. A middleware caps date ranges so one heavy query can't freeze the process.
5. A multi-minute index build runs in a worker thread while the API keeps serving (query went from ~10 s to ~4 ms).
6. Exports are counted before they're built — above 800k rows you get a friendly 413, not a crashed server.

Plus JWT + email OTP login and per-report permissions that drive both the API and the sidebar.

Stack: Express 5, better-sqlite3, PostgreSQL, SQL Server, Redis, BullMQ worker, React 19 with TanStack Query/Table/Virtual. Deployed on Windows IIS.

Unusual choice: SQLite as a read-model on a server. It worked because I respected its one rule — a single writer.

Would you have used SQLite here, or gone straight to a columnar store?

#DataEngineering #NodeJS #SQLite #PostgreSQL #BusinessIntelligence #Performance

---

## Post 3 — WhatsApp Dashboard (real-time)

Two WhatsApp Business numbers, one inbox, and replies must always go out from the number the customer wrote to.

That was the brief for my first real-time app.

How it works:
→ Exotel sends inbound messages and delivery receipts to an Express webhook.
→ I upsert by message ID (webhook retries become harmless) and store the raw payload.
→ Socket.IO pushes the message, unread count and delivery status to every agent's browser.
→ The contact remembers which business number it last used; outbound picks that number automatically.
→ Media goes to Cloudinary — voice notes are transcoded to MP3 because WhatsApp rejected the original format.
→ Reports are MongoDB aggregation pipelines: status breakdown, daily series, top contacts.

Deployment: Cloudways (Linux) with PM2 behind an Apache reverse proxy — including the WebSocket upgrade rule that took me an evening to get right.

Lesson: make webhooks idempotent on day one. You will receive duplicates.

What's your go-to pattern for webhook idempotency?

#MERN #SocketIO #MongoDB #WhatsAppAPI #NodeJS #RealTime

---

## Post 4 — QMS (call-quality audit)

92 seconds → 300 milliseconds. One SQL query. Here's what was wrong.

Context: QA auditors review calls. A transferred call has two legs with different call IDs, and the patient's clinical data lives in another database. They were opening three systems per call.

I built a report: paste a Call ID → both legs, both agents, durations, hangup party, disposition, patient and clinical capture, and the call recording playing right in the page.

The slow part: an official SQL Server view ran a per-row OUTER APPLY. I bypassed it — base tables + UNION ALL + two indexes — and added a per-query timeout with cancellation so a slow field becomes a warning badge instead of a broken page.

The interesting part: recordings live on two servers (recent vs archive). I probe each with a 2-byte ranged GET, detect 0-second stub files, and stream the audio through my backend with HTTP Range support so seeking works. The upstream URL is rebuilt from a whitelist — never from client input.

Rule for reporting tools: never return 500 for missing data. Degrade and explain.

What's your favourite "one index changed everything" story?

#SQLServer #PostgreSQL #NodeJS #Performance #Redis #ReactJS

---

## Post 5 — Patient-CRM (caching)

An agent searched a patient's phone number and saw "no calls". The patient had called 12 times — from a different number.

Problem: patients have up to four phone numbers stored in different columns. Call records only know the number that dialed.

Solution: resolve the patient first, collect all four numbers (plus 0-prefixed variants), then fetch history for all of them — with counts per number so the agent understands.

The performance part — a three-tier cache:
1. In-process LRU (10 min) — same agent, same search, instant.
2. Redis (5 min) — shared across processes.
3. A local SQLite index of the last 30 days, refreshed every 15 minutes from PostgreSQL using a server-side cursor (10k rows per fetch, atomic file swap).
4. PostgreSQL itself — fallback and gap-fill for the newest minutes.

Every tier is optional. If Redis dies, the app gets slower, not broken.

From ~2 minutes to milliseconds.

What's the smartest cache invalidation trick you've used?

#Caching #Redis #PostgreSQL #SQLite #NodeJS #BackendEngineering

---

## Post 6 — Cohorts-Analytics

I ported a Python analytics script to a web app — and the hardest part was matching the numbers exactly.

The tool: cohort retention. Group customers by first-purchase month, track how many buy again in each later month, with revenue and lifetime value.

The build:
→ Mirror 4.5M order rows from SQL Server into SQLite with keyset pagination (WHERE id > last ORDER BY id — constant cost per page).
→ Translate T-SQL → SQLite (FORMAT → substr, DATEDIFF → julianday, ISNULL → COALESCE, and COLLATE NOCASE to mimic SQL Server's case-insensitive collation).
→ Two modes: New-to-Brand and Overall (with a same-month diagonal).
→ Styled Excel export with the same colour heatmap as the screen (ExcelJS, frozen panes).
→ Login with email OTP — the OTP is never stored; an HMAC of it lives inside a 5-minute token.

Validation: I kept a diagnostic script that targets the known totals from the Python tool and iterated until they matched to the rupee.

Lesson: when porting analytics, the spec is the old output.

Have you ported analytics between SQL dialects? What bit you?

#Analytics #SQL #NodeJS #ReactJS #DataEngineering #ExcelJS

---

## Post 7 — CallDropAutoDial

A dropped inbound call is a lost lead. So I built a job that gets them back — safely.

Every 30 minutes during business hours:
1. Query the dialer warehouse (PostgreSQL) for calls that dropped or were abandoned in the window.
2. Exclude customers who already reconnected (NOT EXISTS).
3. One row per phone (DISTINCT ON), then de-duplicate again on the normalised 10-digit number.
4. Route by queue to the right outbound campaign and upload in one batch per campaign.

Safety, because this dials real people:
→ DRY_RUN=true by default — builds and logs the payload, sends nothing.
→ A re-entrancy guard so overlapping cycles skip.
→ One worker process on IIS so the cron can't double-fire.
→ Preview endpoint so operations can see the list before going live.
→ No automatic retries — a retry could double-dial. Log the exact payload instead.

And one surprise: the table in the spec was filled by a 1-hour ETL, so a 30-minute window was always empty. Switched to the live parent table and documented why.

Small service, ~1,000 lines, half of it comments explaining decisions.

What safety switches do you add to automation that touches customers?

#Automation #NodeJS #PostgreSQL #Cron #BackendEngineering

---

## Post 8 — Email-Extract (queues)

I built a scraping pipeline to learn BullMQ properly. Four queues, four workers, one lesson.

Flow: search query → SerpAPI → one job per URL → scrape → validate emails → store.

Design choices:
→ Cheerio first (fast), probe /contact and /about pages, Puppeteer only as a last resort.
→ Per-queue policies: scraper 3 attempts with exponential backoff, concurrency 5; validation concurrency 3.
→ Validation ladder: regex → disposable-domain list → DNS MX lookup → status valid / risky / disposable / invalid.
→ Dedup at three levels: in-memory set → DB check → unique compound index with unordered insertMany.
→ JWT access (15 min) + refresh (7 days) tokens; every query scoped by owner.

Frontend: React 19, Redux Toolkit for auth/UI state, React Query for server state, react-hook-form, Tailwind 4, Recharts.

Still an MVP with a backlog — but the queue design is the part I'd reuse anywhere.

The lesson: put the slow, flaky work behind a queue from day one. The API stays fast and the retries are free.

BullMQ, Bull, or something else — what do you reach for?

#BullMQ #Redis #Puppeteer #MongoDB #NodeJS #WebScraping

---

## Post 9 — Deployment lessons (IIS + Cloudways)

Six Node apps on Windows IIS and one on Linux. Things nobody tells you.

IIS + iisnode:
→ process.env.PORT is a named pipe, not a number. parseInt() → NaN → nothing listens.
→ iisnode only restarts Node when a *watched* file changes. Forget to watch src/ and your deploy silently keeps running old code.
→ App pool: No Managed Code, AlwaysRunning, Idle Time-out 0, Preload — or your cron job dies at night.
→ SQLite-backed app? nodeProcessCountPerApplication = 1. Always.
→ Want no CORS? ARR + URL Rewrite: proxy /api to Node from the same site.

Cloudways + PM2 + Apache:
→ .htaccess [P,L] rule to proxy to 127.0.0.1:5000 — and a separate rule for the WebSocket Upgrade header.
→ pm2 save + pm2 startup, or a reboot forgets your app.
→ max_memory_restart is your friend.

Both: a /health/ready endpoint that pings every datastore and returns 503 when degraded. Verify every deploy from the log line that prints the loaded env file and feature flags.

What's your most painful "it works on my machine" deployment story?

#DevOps #IIS #NodeJS #PM2 #Deployment #WebDevelopment

---

## Post 10 — Wrap-up / lessons

8 apps, ~155,000 lines, 9 months, one developer. What I'd tell myself at the start.

1. One architecture, reused. routes → validation → controller → service → repository. Every project got faster.
2. Generate boilerplate. A DDL-to-model script gave me 463 models and zero typos.
3. Parity before polish when replacing legacy.
4. Read-models beat clever queries when the data is big and the host is small.
5. Make every webhook idempotent, every cache tier optional, every automation dry-runnable.
6. Write the runbook while the pain is fresh.
7. What I'd change: TypeScript from day one, tests per service, secrets out of the repo, CI/CD instead of manual deploys.

I'm now looking for a team where I can bring this end-to-end ownership and learn from stronger engineers. If you're hiring full-stack Node/React developers — or just want to talk shop — my DMs are open.

Thanks for following the series. Which post was most useful?

#FullStackDeveloper #NodeJS #ReactJS #SoftwareEngineering #OpenToWork #CareerGrowth
