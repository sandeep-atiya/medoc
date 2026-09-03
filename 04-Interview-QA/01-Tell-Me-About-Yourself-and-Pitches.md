# Tell Me About Yourself + Project Pitches (spoken answers)

Practise these out loud. Keep the same order every time: who you are → what you built → stack → impact → what you want next.

---

## 1. "Tell me about yourself" (60–90 seconds)

> "I'm [Your Name], a full-stack developer working mainly with Node.js, Express and React, with strong SQL Server, PostgreSQL, MongoDB and Redis experience.
>
> Over the last year I built and deployed eight applications end-to-end for a healthcare tele-consultation company that runs a large call center on the Ameyo dialer. The two biggest are a call-center CRM that replaced a legacy ASP.NET WebForms system — about 78,000 lines with 47 API modules and integrations with the dialer, Exotel WhatsApp and SMS — and a BI reporting platform with 45+ reports over 50-million-row telephony tables, built with an ETL into a SQLite read-model so reports return in milliseconds.
>
> Around those I built a real-time WhatsApp inbox with Socket.IO on MongoDB, a cohort retention dashboard, a call-quality audit tool with in-page recording playback, a cached patient lookup, and a cron-based lead-recovery automation, plus a lead-generation SaaS prototype with BullMQ queues.
>
> In all of them I owned requirements, design, backend, frontend and deployment — most on Windows IIS with iisnode, one on Cloudways with PM2 — and I supported them in production.
>
> I'm now looking for a role where I can work on larger-scale systems with a team, and keep growing in system design and DevOps."

**Shorter version (30 seconds):**
> "Full-stack developer, Node/Express + React, with SQL Server, PostgreSQL, MongoDB and Redis. In the last year I built eight production apps end-to-end for a healthcare call-center business — a CRM replacing a legacy ASP.NET system, a 45-report BI platform, a real-time WhatsApp inbox, analytics and automation tools — and deployed them myself on IIS and Cloudways."

---

## 2. "Walk me through your projects" (2–3 minutes total)

Use this ladder. Say the bold line, then one extra sentence for each.

1. **Appointment-CRM** — "The main agent CRM inside the Ameyo dialer; replaced ASP.NET WebForms; React + Express 5 + SQL Server; 463 auto-generated models; WhatsApp/SMS/dialer integrations; on IIS."
2. **ViewReports** — "The company's BI portal; 45+ reports; ETL from PostgreSQL and SQL Server into SQLite; JWT + OTP; RBAC per report; Excel export guards; on IIS."
3. **WhatsApp Dashboard** — "Real-time WhatsApp inbox over Exotel; webhooks → MongoDB → Socket.IO; multi-number routing; Cloudways with PM2 and Apache proxy."
4. **QMS** — "Call-quality audit by Call ID; reconstructs two call legs; SQL Server + PostgreSQL + Redis; call recording streamed through a Range-aware proxy; a 92-second query brought to 300 ms."
5. **Patient-CRM** — "Agent lookup of call history across all of a patient's numbers; three-tier cache LRU → Redis → SQLite → PostgreSQL."
6. **Cohorts-Analytics** — "Cohort retention and LTV dashboard, ported from Python; SQL Server mirrored into SQLite; styled Excel export; OTP login."
7. **CallDropAutoDial** — "Cron job every 30 minutes that pushes dropped/abandoned callers into Ameyo campaigns; DRY_RUN and single-process safety."
8. **Email-Extract** — "My own SaaS prototype; BullMQ pipeline for search → scrape → validate emails; Puppeteer, Cheerio, MongoDB, JWT refresh tokens."

Close with: "The common thread is the same layered Express architecture and React front-ends, reused so each new project got faster to build."

---

## 3. "Which project are you most proud of and why?"

> "ViewReports, because it was the hardest engineering problem: reporting on 50-million-row tables in a single-process Windows host. I designed a chunked incremental sync into a SQLite read-model, built an index in a worker thread so the API kept serving, added a date-range guard so one heavy report could not freeze the process, and keyed the cache on the last-sync timestamp so invalidation was automatic. Reports went from minutes to milliseconds, and managers stopped depending on one person running SQL by hand."

Alternative (integration-heavy):
> "Appointment-CRM, because I had to read a 250 KB legacy C# code-behind, keep exact parity, generate 463 models from SQL, and make agents log in with zero clicks from inside the dialer iframe — and it's used every day."

---

## 4. "What was the hardest bug you fixed?"

> "In QMS, after a deploy, a config change kept being ignored in production. The logs showed the old code still running. I found that iisnode only restarts the Node process when a *watched* file changes, and the `watchedFiles` list did not include the `src` folder — so an upload silently kept the old process alive and `.env.production` was only read at startup. I fixed the watched-files list, documented the `Restart-WebAppPool` command in the web.config comment, and added a startup log line that prints the value of each feature flag, so any deploy can be verified from the log alone."

Second option:
> "In CallDropAutoDial, the query returned nothing for every 30-minute window. The spec pointed to a child table that is filled by a one-hour ETL, so the last 30 minutes were always empty. I switched to the live parent table and documented the reason next to the SQL."

---

## 5. "What is your biggest technical achievement in numbers?"

- 8 applications, ~155,000 lines, 9 months, solo end-to-end.
- 463 SQL tables → models generated by a script I wrote.
- 45+ reports, 50M+ row source tables, reports from minutes → milliseconds.
- 92 s → 300 ms query optimisation.
- 800,000-row Excel guard, 1,000,000-row CSV guard.
- 4 databases/engines (SQL Server, PostgreSQL, MongoDB, SQLite) + Redis in production.
- 6 apps on IIS + iisnode, 1 on Cloudways + PM2.

---

## 6. "Why are you looking for a change?" (keep positive)

> "I've grown a lot by owning whole systems alone, but I want to work with a team of engineers — code reviews, larger-scale architecture, CI/CD and cloud — and contribute what I learned about shipping and operating production systems."

---

## 7. "What do you want to learn next?"

> "TypeScript across the stack, containerisation and CI/CD (Docker, GitHub Actions), and cloud services like AWS. I have already used managed services — MongoDB Atlas, Redis Cloud, Cloudways — and I want to go deeper."

---

## 8. Elevator pitch for a recruiter call (20 seconds)

> "Full-stack Node.js and React developer with real production experience: I built and deployed eight apps for a healthcare call-center business — CRM, BI reporting, real-time WhatsApp, analytics and automation — on SQL Server, PostgreSQL, MongoDB and Redis, hosted on IIS and Cloudways. I own features from requirements to production support."
