# Project 7 — CallDropAutoDial (Scheduled Call-Back Automation → Ameyo)

**Repo:** `D:\CallDropAutoDial\Backend` (GitHub: CallDropAutoDial) — 2 commits: "application completed", "Live server deployment setup"
**Period:** July → August 2026 (live 18–19 August 2026)
**Size:** ~35 files, ~2,300 lines (23 source files, heavily commented) — backend only, no UI
**My role:** Sole developer — requirement analysis with operations, SQL detection logic, Ameyo API integration (verified live), cron scheduling, safety design, IIS runbook.

---

## 1. Pitches

**One line:**
> A headless Node.js service that every 30 minutes during business hours finds customers whose calls dropped or were abandoned, de-duplicates them, and uploads them into the correct Ameyo outbound campaign so an agent calls them back automatically.

**30-second pitch:**
> "When an inbound call drops or a customer abandons the queue, that lead was simply lost. I built a small Express service with a node-cron job: every 30 minutes from 9 AM to 7 PM it runs a PostgreSQL query on the Ameyo ACD table to find dropped and abandoned calls in the last window, excludes customers who already reconnected, normalises the numbers to 10 digits, and posts them in batches to Ameyo's uploadContacts API — routing each queue to the right campaign and lead list. It has a DRY_RUN switch, a re-entrancy guard, a manual trigger endpoint and a preview endpoint, and it runs on IIS as exactly one process so the cron never double-fires."

---

## 2. Business problem

- Dropped/abandoned inbound calls = lost sales leads; agents had no list to call back.
- The old approach was an ASP.NET page someone ran manually.
- Different queues (verification vs fresh) must go to different outbound campaigns.

## 3. What I built — features

- **Detection SQL (PostgreSQL):** `SELECT DISTINCT ON (phone)` over the live ACD table with validity filters; `drop_type` = `CALL_DROP` (disposition Call Drop) or `ABANDONED` (no disposition + system hangup); time window; **`NOT EXISTS` reconnect-exclusion** (a later successful call for the same phone within the window removes it); ordered latest-first per phone; matches a partial index.
- **Routing:** queue → campaign id + lead id mapping from env (verification queue → utilisation campaign; fresh queue → NC campaign).
- **Two-stage dedup:** SQL `DISTINCT ON`, then in-memory dedup on the normalised 10-digit number (so `0941…` and `0091941…` collapse), keeping the latest call time.
- **Ameyo upload:** `command=uploadContacts` as `application/x-www-form-urlencoded` with a JSON `data` field; headers `requesting-host`, `hash-key`, `policy-name`; one batch POST per campaign; one campaign failing never blocks the other; per-record result accounting (inserted count, result buckets, first 10 rejects logged).
- **Scheduling:** `*/30 9-18 * * *` plus a final run at 19:00, timezone Asia/Kolkata.
- **Safety:** `DRY_RUN=true` by default (builds and logs the payload, does not POST); module-level `running` flag → overlapping cycle returns `{skipped:true}`; deliberate no-retry, log-only policy (agreed with seniors); fail-fast config with envalid; DB ping before listen; graceful shutdown with a 10 s forced-exit timer.
- **Endpoints:** `GET /api/health` (503 if DB down), `GET /api/call-drops?minutes=N` (preview), `POST /api/job/run` (manual trigger behind a strict rate limit).
- **Docs:** `AMEYO_API.md` (verified live: e.g., auth failure returns literal `null` with HTTP 200; response is `text/plain` containing JSON) and `DEPLOY_IIS.md` runbook.

## 4. Architecture

```
node-cron (2 schedules, Asia/Kolkata)  ┐
POST /api/job/run (manual)             ├─► runCallDropCycle(trigger)  [re-entrancy guard, DRY_RUN]
npm run job:once (one-shot script)     ┘         │
                                                 ▼
                    callDrop.repository (pg Pool, parameterised CALL_DROP_SQL)
                                                 ▼
                    callDrop.service: attach routing, normalise phones, dedup
                                                 ▼
                    ameyo.service: group by campaign → uploadContacts batch POST → per-record accounting
                                                 ▼
                    Winston logs (app.log / error.log, rotation) + JSON cycle summary
IIS + iisnode: 1 worker process, AlwaysRunning, idle timeout 0, no recycling
```

## 5. Exact tech stack

Node.js ≥18 (ESM), Express 5.2, pg 8, node-cron 4, envalid, dotenv, axios, dayjs, Winston, helmet, compression, express-rate-limit; ESLint/Prettier/nodemon. No frontend, no ORM, no auth (server-side/curl consumption only; CORS intentionally not installed yet).

## 6. Deployment (documented runbook)

- Physical path on the Windows server; IIS site + dedicated app pool; binding on a dedicated port; test URL `/api/health`.
- App pool: No Managed Code, AlwaysRunning, Idle Time-out 0, recycling interval 0, **Max Worker Processes 1** ("more = duplicate pushes to Ameyo"), Rapid-Fail off; site Preload enabled; `appcmd` one-liners included.
- `web.config`: iisnode handler, `nodeProcessCountPerApplication="1"`, `node.exe --max-old-space-size=4096`, `enableXFF` + `trust proxy`, logging to `logs\iisnode`.
- `PORT` kept as string so it can be a TCP port locally or an iisnode named pipe live.
- Deploy = `npm ci --omit=dev`; flip `DRY_RUN=false` to go live.

## 7. Challenges and solutions

1. **Spec said use `acd_call_details`, but it was always empty for the last 30 minutes.** → Found it is filled by a 1-hour ETL; switched to the live parent table `acd_interval_denormalized_entity`. Documented the correction in code.
2. **Ameyo returns HTTP 200 with `null` on auth failure and `text/plain` JSON.** → Custom `transformResponse` + manual parse; explicit null check; documented in `AMEYO_API.md` after live tests.
3. **Same customer in different formats.** → `normalizePhone()` = digits only, last 10 (matches legacy `RIGHT(mobile,10)`); Ameyo rejects other lengths (`PREPROCESSING_INVALID_LENGTH`).
4. **Risk of double-dialing customers.** → single IIS process, re-entrancy guard, DISTINCT ON + in-memory dedup, reconnect exclusion, DRY_RUN before go-live.
5. **Who gets alerted on failure?** → Log-only policy per seniors; failing payload logged in full for diagnosis.

## 8. Interview questions specific to this project

**Q: Why not retry failed uploads?**
A: A retry could double-upload if the first request actually succeeded server-side (Ameyo responses are not always clear). The agreed policy was log-only with the exact payload so operations can re-run manually via `/api/job/run`.

**Q: How do you make sure the cron does not run twice?**
A: Three layers: one Node process in IIS, an in-process `running` flag that skips overlapping cycles, and idempotent SQL windows.

**Q: How would you scale this to multiple campaigns?**
A: The queue→campaign mapping is config-driven; add an entry per queue. Grouping by campaign already produces one batch per campaign.

**Q: Why Express at all for a cron job?**
A: Health endpoint for IIS/monitoring, preview endpoint for operations to check what will be dialed, and a manual trigger — plus iisnode requires an HTTP entry point.

**Q: What does `DISTINCT ON` do?**
A: PostgreSQL-specific: keeps the first row per phone according to the ORDER BY (latest call first), so one row per customer.

## 9. Be careful — what NOT to claim

- No frontend, no authentication, no CORS. It is an internal server-side job.
- No retries by design. Do not say "retries with backoff" for this project.
- Joi is installed but the validator is hand-written.

## 10. What I would improve next

- API key on the manual trigger; small status page.
- Persist cycle summaries in a table for reporting; alerting via email/Slack once approved.
- Idempotency ledger of uploaded numbers per day.

## 11. Resume bullets

- Built and deployed a scheduled lead-recovery service (Node.js/Express 5, PostgreSQL, node-cron) that detects dropped and abandoned inbound calls from the Ameyo ACD warehouse every 30 minutes, de-duplicates and normalises numbers, and batch-uploads them to the correct Ameyo outbound campaigns via the uploadContacts API.
- Engineered for safe automation: DRY_RUN mode, re-entrancy guard, reconnect-exclusion SQL, single-process IIS deployment, fail-fast env validation, structured Winston logging, and a live-verified Ameyo API reference and IIS runbook.

## 12. Keywords

automation, cron, node-cron, PostgreSQL, DISTINCT ON, NOT EXISTS, Ameyo API, batch upload, deduplication, idempotency, envalid, Winston, Express 5, IIS, iisnode, runbook
