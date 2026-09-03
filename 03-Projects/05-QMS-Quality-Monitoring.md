# Project 5 — QMS (Quality Monitoring System: Call-Quality Audit Report)

**Repo:** `D:\qms\qms` (nested folder) — 3 commits, latest 2 September 2026
**Period:** July → September 2026
**Size:** ~65 source files, ~2,400 lines + 477-line SQL setup + 423-line spec I wrote
**My role:** Sole developer — wrote the build spec/data dictionary with the QA team, designed the two-leg call algorithm, PostgreSQL + SQL Server queries and indexes, Ameyo recording integration and streaming proxy, React page, IIS deployment.

> Note: here QMS means **Quality Monitoring / Quality Management**, not "queue management". Say "call-quality monitoring report".

---

## 1. Pitches

**One line:**
> A call-quality audit tool: a QA auditor enters one Call ID and gets the complete audit sheet — both call legs, both agents, durations, hangup party, disposition, patient details, clinical capture fields, NTB/Repeat flag and an in-page playable call recording.

**30-second pitch:**
> "QA auditors used to open the dialer console, then two databases, to review a single call. I built a report that takes a Call ID, reconstructs the whole interaction from the PostgreSQL dialer table — because a transferred call has two legs with different call IDs but the same interaction ID — enriches it with patient and clinical data from SQL Server, and finds the recording from Ameyo's live or archive server. The recording streams through my backend with HTTP Range support so it plays in the page. I optimised a SQL Server query from about 92 seconds to under 350 ms by bypassing a slow view, cached reports in Redis with versioned keys, and deployed on IIS."

---

## 2. Business problem

- A monitored call often has **two legs**: the original inbound leg (agent A who transferred) and the transferred leg (sales agent B who dispositioned). Auditors had to manually match them.
- Patient identity, medicine-due status and relief % live in SQL Server, not in the dialer.
- Recordings live on two Ameyo servers (live: last ~7–10 days; archive: up to ~3 years).
- Nobody had one screen with everything.

## 3. What I built — features

- `GET /api/reports/qms-call-report?callId=` → 16-field report: call ID, interaction ID, transferred yes/no, campaign(s), date/time(s), phone (masked), patient (id, name, city, state), call duration, transferred duration, hangup by, agent + transferred agent, agent disposition, NTB/Repeat flag, due captured, relief status/percent captured, recordings, both legs raw, sources used, `warnings[]`, `fromCache`.
- `GET /api/reports/qms-call-report/recording?source=live|archive&id=` → audio stream proxy with Range support and download option.
- Health: `/api/health/live` and `/api/health/ready` (pings PostgreSQL, SQL Server, Redis → 200 or 503).
- React page: Call-ID search, summary card, recordings card with tiles, 4 KPI cards, patient card with NTB/Repeat badge, "captured during call" card, agent cards, two leg panels, amber warning badges. Embed mode when `?callId=` is in the URL (used inside ViewReports).
- Docs: README with core logic and assumptions to confirm, a 423-line spec/data dictionary, SQL index scripts for both databases marked idempotent for go-live, `test:connections` script.

## 4. Architecture / core algorithm

```
callId ─► PostgreSQL dialer_call_details: fetch leg by call_id ─► read crt_object_id
        ─► fetch all legs with that crt_object_id
        ─► classify: transferred leg = call_type 'transferred.to.campaign.dial' OR association_type 'transfer...'
                     original leg  = earliest non-transferred leg
        ─► map each report field to the correct leg (durations, agents, disposition, hangup)
        ─► SQL Server: patient by phone (variants), enrichment by patient id (medicine due, relief),
                       order count → NTB (≤1 order) vs Repeat
        ─► Ameyo: probe live voice-log by crt_object_id; if no audio, probe archive by call_id
        ─► Redis cache (key qms:callreport:v5:<callId>, 5 min TTL)
        ─► JSON with warnings[] (never 500 for missing data; 404 only if call ID exists in no leg)
```

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 8 + React Compiler, TanStack Query 5, Tailwind CSS 4 (CSS-first config), shadcn-style badge/button/card (cva + clsx + tailwind-merge), lucide-react, sonner, axios, Geist/Inter fonts |
| Backend | Node.js ESM, Express 5, envalid, Winston + morgan, helmet, cors, compression, express-rate-limit, native `fetch` for Ameyo |
| Data | PostgreSQL (pg) `dialer_call_details`; SQL Server (mssql 12) `tblPatient`, `MedicineDueDetails(2)`, `PatientReliefPercent(2)`, `tblOrderDetails`; Redis (ioredis) |
| Deploy | IIS + iisnode (`QmsPool`, 2 processes, detailed `watchedFiles`), `.env.development/.staging/.production`, `vite build --mode production` |

## 6. Notable technical implementations (know these well)

1. **Recording availability probe:** a ranged `GET bytes=0-1` request, body cancelled, returns tri-state `available / missing / unknown`. Detects Ameyo's 0-second stub files (below 2048 bytes) and HTML/JSON error pages served with HTTP 200 by sniffing content-type. `unknown` (server unreachable) still shows the link, marked unverified, because an unreachable server proves nothing.
2. **Audio streaming proxy:** upstream URL is rebuilt from a whitelisted base (never from client input) → no open proxy; IDs regex-validated; forwards `Range`, relays `content-range`/`accept-ranges` so the `<audio>` element can seek; aborts upstream when the client disconnects; 20 s connect timeout; `Cross-Origin-Resource-Policy: cross-origin` only for this route.
3. **Ameyo URL encoding gotcha:** the `data={'crtObjectId':'…','targetFormat':'mp3'}` parameter must be percent-encoded (including `'` → `%27`) or Ameyo returns HTTP 500; handled identically on server and client.
4. **Per-query SQL Server timeout with cancellation:** `runQuery()` races the query against a 12 s timer and calls `request.cancel()`; on timeout the field becomes `null` + a specific warning instead of an error.
5. **View bypass:** the official view ran a per-row `OUTER APPLY`; querying base tables with `UNION ALL` + new indexes took the query from ~92 s to ~200–350 ms.
6. **Versioned cache key** (`v2 → v5`) with a changelog so schema changes never serve stale cached reports.
7. **Express 5 specifics:** `req.query` is a read-only getter in Express 5, so validation writes to `req.callId`; `xss-clean`/`hpp` intentionally not mounted because they reassign `req.query` and throw under Express 5.
8. **Rate limit sized for reality:** 12,000 requests/minute because ~50 auditors share one office IP.
9. **iisnode lesson:** without `src\...` entries in `watchedFiles`, an upload silently keeps running old code and env changes seem ignored; documented the fix and the `Restart-WebAppPool` command.

## 7. Challenges and solutions

1. **Matching two legs of one call.** → Discovered `crt_object_id` repeats across legs; built the classification rules; verified with the QA team on live calls.
2. **92-second query.** → Analysed the view, bypassed it, added indexes (script marked "re-run on production before go-live"). Result ~300 ms.
3. **Recording sometimes exists only in archive, sometimes is a 0-second stub.** → Probe logic with size and content-type checks; archive only contacted when live has nothing; archive can be switched off by env when it is slow (logged at startup so deploys are verifiable from logs).
4. **Must never break the auditor's page.** → "Never-500" contract: degrade to `null` + warning badge.
5. **Config change ignored after deploy.** → iisnode `watchedFiles` fix + runbook.

## 8. Interview questions specific to this project

**Q: Why proxy the audio through the backend instead of linking directly?**
A: The Ameyo servers are internal, require specific URL encoding, and I wanted playback inside the page with seeking and a download option — plus I can whitelist the upstream and hide the internal hosts.

**Q: How do you support seeking in the audio player?**
A: Forward the client's `Range` header upstream and relay `Content-Range` / `Accept-Ranges` / 206 status back.

**Q: Why Redis cache with a version in the key?**
A: When the report shape changes, old cached JSON would break the UI. Bumping `v4 → v5` makes old keys unreachable; they expire naturally by TTL.

**Q: What is NTB vs Repeat?**
A: New-to-Brand = first order; Repeat = more than one order. Threshold is a constant flagged "confirm with business" and documented in the README.

**Q: How do you handle masked phone numbers in SQL Server?**
A: Dynamic Data Masking masks SELECT output but not WHERE predicates, so lookups still match; the app detects masked values and logs that the login needs UNMASK; the UI masks numbers anyway for privacy.

**Q: Where is authentication?**
A: Not in this service — it is embedded via iframe inside the authenticated ViewReports portal on the LAN. Next step is sharing the JWT middleware.

## 9. Be careful — what NOT to claim

- No authentication inside QMS itself (explicit code comments). Explain the host-portal model.
- `bullmq`, `node-cron`, `nodemailer`, `multer`, `swagger`, `jsonwebtoken` are installed but unused; there are no background jobs here.
- The archive recording source is **disabled by env** in production because the archiver is slow; say "supported and switchable".
- No SQLite in QMS (that is in Patient-CRM and ViewReports).

## 10. What I would improve next

- Shared JWT auth + audit log of who viewed which call.
- Batch audit mode (list of call IDs → Excel).
- Cache the recording probe result separately with a longer TTL.

## 11. Resume bullets

- Designed and built a call-quality audit report (Express 5 + React 19) that reconstructs multi-leg dialer interactions from PostgreSQL, enriches them with clinical data from SQL Server, and streams Ameyo call recordings through a Range-aware, whitelist-hardened proxy for in-page playback.
- Cut a critical SQL Server query from ~92 s to ~300 ms by bypassing a per-row OUTER APPLY view and adding indexes; added per-query timeouts with cancellation, versioned Redis caching and a "never-500" degradation contract with user-visible warnings.

## 12. Keywords

call quality monitoring, QA audit, PostgreSQL, SQL Server, Redis caching, Express 5, React 19, TanStack Query, Tailwind 4, audio streaming, HTTP Range, proxy hardening, query optimisation, indexes, envalid, Winston, IIS, iisnode, Ameyo
