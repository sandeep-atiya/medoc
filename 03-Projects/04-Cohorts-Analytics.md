# Project 4 — Cohorts-Analytics (Customer Retention & LTV Dashboard)

**Repo:** `D:\Cohorts-Analytics` (GitHub: Cohorts-Analytics)
**Period:** June 2026 (6 commits, ported from an existing Python tool)
**Size:** ~24 source files, ~2,700 lines of logic + a production SQLite mirror (~900 MB)
**My role:** Sole developer — ported the analytics from Python to Node, designed the MSSQL → SQLite mirror + sync, cohort SQL, OTP login, Excel export, IIS + ARR deployment.

---

## 1. Pitches

**One line:**
> A cohort retention and lifetime-value dashboard that mirrors the order table from SQL Server into SQLite, computes monthly cohort retention heatmaps and KPIs, and exports styled Excel — with JWT + email OTP login.

**30-second pitch:**
> "The business had a Python script for cohort analysis that only one person could run. I rebuilt it as a web app: an Express 5 backend mirrors ~4.5 million order rows from SQL Server into a local SQLite database using keyset pagination, computes cohort retention in two modes — New-to-Brand and Overall — and caches results in summary tables. The React dashboard shows KPI cards (total customers, repeat rate, average LTV, average days to repeat), a colour heatmap, and exports CSV or styled Excel that looks exactly like the screen. Login is JWT with email OTP, and it is deployed on IIS with ARR reverse proxy. The numbers were validated against the original Python output."

---

## 2. Business problem

- Marketing/management wanted to know: of customers who first bought in month X, how many bought again in later months, and what revenue they brought (retention + LTV).
- The existing Python script was slow, manual, and not shareable.
- The order table is large (~4.5M rows) and lives on the production SQL Server, which should not be hammered with analytics queries.

## 3. What I built — features

- **Two-phase sync (daily cron at 2 AM + manual refresh):**
  1. Raw sync: SQL Server → SQLite with keyset pagination (`WHERE id > @last ORDER BY id`, 50k rows per page, `NOLOCK`), merging the live table and a frozen backup table, dedup with `INSERT OR IGNORE` on the primary key.
  2. Aggregation: five SQL queries run locally in SQLite and results are written into cache tables in one transaction.
- **Cohort logic:** delivered orders only; cohort month = first delivered order month; same-day duplicate orders collapsed with `ROW_NUMBER()`; revenue = sum of delivered amounts.
- **Two modes:** `ntb` (fixed cohort base) and `overall` (all customers per calendar month, with a same-month "diagonal" for repeat-in-same-month).
- **Dashboard:** 4 KPI cards, filters (year/month/date range/mode), heatmap table with 6 colour bands.
- **Exports:** CSV (with UTF-8 BOM so Excel opens it correctly), 2-sheet Excel, styled single-sheet Excel (frozen panes, colours identical to UI, multi-line count + ₹ cells, ₹ Cr/L/K formatting), per-cohort customer list Excel (repeated / not repeated).
- **Auth:** login → OTP email → JWT. OTP is not stored; an HMAC of the OTP is embedded in a 5-minute temp token and verified with a constant-time comparison. Password check is compatible with the legacy ASP.NET encryption.
- **Ops:** cross-process sync lock with 30-minute staleness, schema self-migration for cache tables, `sync-status` endpoint polled by the UI during refresh.

## 4. Architecture

```
SQL Server (tblOrderDetails live + frozen backup)  ── read-only, NOLOCK, keyset pages of 50k
             ▼
Express 5  jobs/rawSync.js  →  SQLite mirror (better-sqlite3, WAL, NOCASE collation, 3 composite indexes)
           jobs/syncJob.js  →  5 aggregation queries  →  cohort_summary / cohort_monthly_retention /
                                                        dashboard_summary / overall_* / sync_metadata
             ▼
API (/api/v1): auth (login, verify-otp, me), cohort (summary, report, refresh, sync-status, exports)
             ▼
React 19 SPA: LoginPage → OtpVerificationPage → CohortPage (SummaryCards, CohortFilters, CohortTable)
IIS: site on port A serves SPA; ARR + URL Rewrite proxies /api/* → Node on port B (iisnode)
```

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 8 + React Compiler, react-router-dom 6, Context + useReducer, custom hooks, hand-written CSS + Tailwind 4 setup, lucide-react, axios |
| Backend | Node.js ESM, Express 5, better-sqlite3 (WAL), mssql, node-cron, nodemailer (Gmail SMTP), jsonwebtoken, ExcelJS, Joi, helmet, cors, compression, morgan |
| Deploy | IIS + iisnode + ARR reverse proxy; `start.bat`/`stop.bat` for local dev |

## 6. Key SQL / data notes

- Dialect translation MSSQL → SQLite documented in code: `FORMAT(date,'yyyy-MM')` → `substr(date,1,7)`; `DATEDIFF(DAY,a,b)` → `CAST(julianday(b)-julianday(a) AS INTEGER)`; `ISNULL` → `COALESCE`.
- Case-insensitive matching mirrored with `COLLATE NOCASE` for `disposition_Code` and `CourierRemark`.
- Overall mode materialises a `TEMP TABLE` + index to avoid re-scanning 4.5M rows through non-materialised CTEs.
- Retention matrix = `UNION ALL` of cross-month joins + same-month diagonal.

## 7. Deployment

- Backend `web.config` for iisnode (1 process, 4096 concurrent requests, logs).
- Frontend `public/web.config`: SPA fallback, MIME maps, `nosniff`, and **ARR reverse-proxy rule** `/api/*` → `http://localhost:<backendPort>/api/*` so the SPA and API share one origin (no CORS issues, one URL for users).
- Env loading uses an absolute path because iisnode's working directory is not the project root; PORT may be a named pipe.

## 8. Challenges and solutions

1. **Numbers had to match the Python tool exactly.** → Ported the MSSQL queries faithfully, wrote a diagnostic script targeting the known values (avg LTV 3846.91, total customers 1,366,125), fixed collation and date-format differences until they matched.
2. **Production SQL Server must not be loaded.** → Read-only keyset pagination with NOLOCK at 2 AM into a local mirror; all analytics run on SQLite.
3. **Two source tables (live + frozen backup) overlap.** → Load live first, then `INSERT OR IGNORE` from backup so live wins.
4. **Excel export must look like the screen.** → ExcelJS with ARGB colour bands, frozen header/first column, rich multi-line cells.
5. **Concurrent refresh clicks.** → Single-row `sync_lock` table; atomic insert as mutex; 30-minute staleness reset.

## 9. Interview questions specific to this project

**Q: What is a cohort analysis?**
A: Group customers by the month of their first purchase (cohort), then track what percentage of each cohort buys again in each following month. It shows retention and how it changes for newer cohorts.

**Q: Why not compute directly on SQL Server?**
A: Production load and speed. The mirror lets me run heavy window functions locally and re-run instantly with different filters.

**Q: How is the OTP secure if it is not stored?**
A: The server returns a short-lived JWT containing an HMAC-SHA256 of the OTP keyed with the server secret. Verification recomputes the HMAC and compares in constant time. The OTP itself only goes to the email.

**Q: What is keyset pagination and why use it over OFFSET?**
A: `WHERE id > lastId ORDER BY id LIMIT n` — each page starts from the last key, so cost stays constant; OFFSET scans and discards rows and gets slower on every page.

**Q: How would you handle a table 10× bigger?**
A: Incremental sync by `id` only (already keyset), partition the mirror by year, or move the mirror to DuckDB/ClickHouse for columnar analytics.

## 10. Be careful — what NOT to claim

- package.json description mentions PostgreSQL, Redis and BullMQ, but **none are used**. The stack is SQL Server + SQLite + node-cron only.
- Recharts is installed but the heatmap is a plain HTML table.
- The cohort API routes are **not protected** by the JWT middleware (only auth routes are). If asked: "the UI is behind OTP login; adding the auth middleware to cohort routes is a one-line improvement I have noted" — do not claim they are protected.

## 11. What I would improve next

- Apply auth middleware to cohort routes; pass the token on export downloads.
- Incremental daily sync instead of a full mirror rebuild.
- Chart library for trend lines; scheduled email of the monthly heatmap.

## 12. Resume bullets

- Ported a Python cohort-analysis tool to a Node.js/React web app: mirrored 4.5M SQL Server order rows into SQLite via keyset pagination, implemented NTB/overall cohort retention and LTV SQL, and delivered a heatmap dashboard with CSV and pixel-matched styled Excel exports (ExcelJS).
- Added JWT + stateless email-OTP two-factor login compatible with legacy ASP.NET password encryption; deployed on IIS with iisnode and ARR reverse proxy; validated outputs against the original tool.

## 13. Keywords

cohort analysis, retention, LTV, analytics dashboard, SQL Server, SQLite, better-sqlite3, keyset pagination, window functions, ETL, node-cron, ExcelJS, OTP, 2FA, JWT, HMAC, React 19, Express 5, IIS, ARR reverse proxy
