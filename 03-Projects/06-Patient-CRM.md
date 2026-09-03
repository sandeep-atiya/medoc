# Project 6 — Patient-CRM (Agent Call-History Lookup with 3-Tier Cache)

**Repo:** `D:\patient-crm\patient-crm` (nested folder)
**Period:** May 2026 → August 2026
**Size:** ~71 source files, ~2,500 lines
**My role:** Sole developer — data model across three databases, 3-tier cache design, background ETL job, React search UI, IIS deployment. This project became the **template** that QMS copied.

---

## 1. Pitches

**One line:**
> A fast lookup tool for call-center agents: type a phone number or patient ID and see the patient's last-30-day call history across all four of their phone numbers, served through an in-memory → Redis → SQLite → PostgreSQL cache chain.

**30-second pitch:**
> "Agents were searching one phone number and seeing no history, because the patient had called from a different number stored in another column. I built Patient-CRM: it resolves the patient in SQL Server, collects all four numbers (mobile, alternate, mobile3, WhatsApp), and returns the call history from the Ameyo warehouse across all of them with per-number counts. To keep it fast, I built a three-tier cache: an in-process LRU, then Redis, then a local SQLite index that a background job fills from PostgreSQL every 15 minutes using a server-side cursor, and only then the live PostgreSQL query. It is embedded as an iframe in the dialer and deployed on IIS."

---

## 2. Business problem

- Patients have multiple phone numbers; call records are keyed by the number that actually called.
- Searching by one number missed most history → agents thought a repeat patient was new.
- The dialer warehouse table is huge; direct queries per agent search were slow (up to 2 minutes).

## 3. What I built — features

- `GET /api/v1/callcenter/customer-search?q=<phone|patientId>` (Joi-validated, 3–50 chars).
- Patient banner, per-number summary with counts, KPI cards, paginated call table (agent, campaign, result, disposition, talk time, direction, remarks).
- Phone-variant matching: 10-digit and `0`-prefixed 11-digit variants for every number; results tagged back to the owning column.
- Phone masking in the UI (`98XXXXXX28`) mirroring SQL Server dynamic data masking.
- Embed/deep-link mode via `?q=` (iframe inside the dialer); request-race protection in the hook (stale responses discarded).
- Background ETL job: full build with a PostgreSQL server-side cursor (`FETCH 10000` batches, `statement_timeout = 0`) into a temp SQLite file then atomic rename; incremental sync every 15 min; deletes rows older than 30 days; VACUUM when needed; sync state in a JSON file.
- Graceful degradation: SQL Server and Redis failures are warnings; only PostgreSQL is required at boot.
- Tiered timeouts: PG direct 120 s, gap-fill 15 s, axios 45 s, with three distinct error messages (timeout / unreachable / server error).

## 4. Architecture

```
Agent (dialer iframe, ?q=) → React SPA (IIS, frame-ancestors *) → axios (45 s timeout)
        ▼
Express 5 API  →  callcenter.service.getCustomerCallHistory(q)
   1. L1 in-process lru-cache (2000 keys, 10 min)
   2. L2 Redis (5 min, key prefix callcenter:search:)
   3. SQL Server tblPatient → collect mobile / alter_mob / Mobile3 / whatsappNo (+ variants)
   4. SQLite phone_call_index (local, WAL) ← built/refreshed by buildSqliteCache.job (PostgreSQL cursor, every 15 min)
   5. PostgreSQL acd_interval_denormalized_entity (fallback + gap-fill for newest rows)
   cache only on success and only when calls.length > 0
```

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 8 + React Compiler, Tailwind CSS 3.4, axios |
| Backend | Node.js ESM, Express 5, Joi, helmet, cors, express-rate-limit (custom keyGenerator for iisnode), morgan, lru-cache, ioredis, mssql 11, pg 8, sqlite3 |
| Deploy | IIS + iisnode (`PatientCallPool`, 2 processes, `watchedFiles .env;web.config`), SPA `web.config` with `frame-ancestors *`, `.env.production` |

## 6. Data

- **SQLite `phone_call_index`** (owned by this app): `call_id` PK, phone, call_date, agent, campaign, result, disposition, talk_time, direction, sub_disposition, remarks; indexes on phone and call_date; WAL + `synchronous=NORMAL`.
- **PostgreSQL** `acd_interval_denormalized_entity` (read-only): ch_call_id, ch_phone, ch_date_added, username, campaign_name, ch_call_result, udh_disposition_code, udh_notes, udh_talk_time, ch_is_outbound.
- **SQL Server** `tblPatient` (read-only): ptID, name, mobile, alter_mob, Mobile3, whatsappNo, city, state.

## 7. Challenges and solutions

1. **Agents saw "no history" for repeat patients.** → Multi-number resolution + variants. Result: complete history in one search.
2. **PostgreSQL search took up to 2 minutes.** → Local SQLite index refreshed every 15 minutes + Redis + LRU. Result: repeat searches in milliseconds, first search in well under a second.
3. **Full index build on a 50M-row table blew memory.** → Server-side cursor with 10k-row fetches, write to a temp file, atomic rename so readers never see a half-built DB.
4. **Sync job crashed the whole server once.** → Stopped exiting on `unhandledRejection`; treat `ECONNRESET`/`EPIPE` as recoverable; documented why.
5. **Rate limiter threw `ERR_ERL_UNDEFINED_IP_ADDRESS` behind iisnode.** → `trust proxy` + custom `keyGenerator` fallback.
6. **Masked phone columns.** → Compare raw columns in WHERE (masking does not affect predicates); detect masked output and log the UNMASK requirement.

## 8. Interview questions specific to this project

**Q: Why three cache levels?**
A: Each solves a different problem: LRU is free and instant for the same agent repeating a search; Redis shares results across the two Node processes; SQLite makes the *first* search fast by pre-indexing the last 30 days locally; PostgreSQL is the source of truth for anything newer than the last sync (gap-fill).

**Q: Why cache only when results are non-empty?**
A: An empty result may be due to a sync gap or a timeout; caching it would hide new calls for 10 minutes.

**Q: How do you keep the SQLite index fresh without downtime?**
A: Full build writes to `.tmp` then renames atomically; incremental sync inserts rows newer than the last sync time every 15 minutes and prunes rows older than 30 days.

**Q: What is a server-side cursor and why use it?**
A: `DECLARE ... CURSOR` keeps the result on the PostgreSQL server and `FETCH 10000` pulls batches, so Node never holds millions of rows in memory.

**Q: How do you avoid showing a stale response when the agent types fast?**
A: A monotonically increasing `requestId` ref; a response is ignored if its id is not the latest.

## 9. Be careful — what NOT to claim

- No authentication (open API on the LAN, embedded in the dialer). Present JWT as next step.
- The "Dashboard" tab is a placeholder ("coming soon"); order history is coded but commented out.
- Many frontend libraries (React Query, Radix, zustand, recharts…) are in package.json but **only axios/react are used**. Do not list them for this project.
- Scheduler is `setInterval`, not node-cron.

## 10. What I would improve next

- JWT auth shared with the host portal; audit log.
- Enable order history and a real dashboard.
- Redis-backed sync state instead of a JSON file; metrics endpoint.

## 11. Resume bullets

- Built a call-history lookup service (Express 5 + React 19) that resolves a patient's four phone numbers in SQL Server and returns 30-day call history from a 50M-row PostgreSQL warehouse through a three-tier cache (in-process LRU → Redis → SQLite index → PostgreSQL), reducing search time from ~2 minutes to milliseconds.
- Implemented a memory-safe ETL using PostgreSQL server-side cursors with atomic SQLite file swaps and 15-minute incremental syncs; hardened for iisnode (named-pipe ports, trust proxy, crash-resistant error handling); embedded as an iframe tool in the dialer.

## 12. Keywords

caching strategy, LRU, Redis, SQLite, PostgreSQL cursor, ETL, incremental sync, SQL Server, dynamic data masking, Express 5, React 19, Tailwind, iframe embedding, IIS, iisnode, performance
