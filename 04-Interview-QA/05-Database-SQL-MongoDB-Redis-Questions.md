# Database Interview Questions — SQL Server, PostgreSQL, MongoDB, SQLite, Redis

You have real production experience with four database engines plus Redis. Answer with the concept, then the project example.

---

## General SQL

**Q1. Explain indexes. When do they hurt?**
An index is a sorted structure (B-tree) that lets the engine find rows without scanning the table. They cost write time and disk. *Examples:* composite indexes on the SQLite read-model (call_time + campaign + agent); a **partial index** `WHERE disposition_code = 'failed.association'` so the index is small (10 s → 4 ms); in QMS I deliberately did **not** index `crt_object_id` and used a bounded time window instead, after measuring real data.

**Q2. What is a covering index?**
An index that contains all columns a query needs, so the engine never touches the table. The CallDropAutoDial query filters were chosen to match an existing partial index on the dialer table.

**Q3. Parameterised queries and SQL injection?**
Values are sent separately from the SQL text, so input can never change the query structure. All my SQL Server (`request.input`) and PostgreSQL (`$1, $2`) queries are parameterised; Joi validates shape before that.

**Q4. Window functions you used?**
`ROW_NUMBER() OVER (PARTITION BY ptId, OrderDate ORDER BY id)` to collapse same-day duplicate orders (Cohorts); `DISTINCT ON (phone) … ORDER BY phone, date DESC` in PostgreSQL to keep the latest call per number (CallDropAutoDial).

**Q5. What is `NOT EXISTS` and when is it better than `NOT IN`?**
An anti-join; `NOT IN` breaks with NULLs and can be slower. I used `NOT EXISTS` to exclude customers who already reconnected within the window.

**Q6. UNION vs UNION ALL?**
UNION removes duplicates (sort cost); UNION ALL keeps all rows. I used UNION ALL to merge `MedicineDueDetails` + `MedicineDueDetails2` (QMS) and to build retention matrices (Cohorts).

**Q7. Explain keyset pagination vs OFFSET.**
Keyset: `WHERE id > @last ORDER BY id` — constant cost. OFFSET reads and discards rows so page 1000 is slow. Used to mirror 4.5M rows in 50k pages.

**Q8. Transactions — where did you use them?**
SQLite `db.transaction()` to write all cohort cache tables atomically; MongoDB `insertMany` with `ordered:false` (not transactional, but resilient to duplicates); upserts by key for idempotency.

**Q9. How do you find a slow query?**
Look at the execution plan (SQL Server: actual plan; PostgreSQL: `EXPLAIN ANALYZE`), find scans and per-row operators, check indexes, rewrite. Real case: SQL Server view with per-row `OUTER APPLY` → bypassed view, base tables + indexes, 92 s → 300 ms.

**Q10. Views vs base tables?**
Views hide complexity but can hide cost; a view with per-row correlated subqueries is dangerous under load. I documented why QMS bypasses the official views.

---

## Microsoft SQL Server

**Q11. How do you connect from Node?**
`mssql` package (tedious driver), `ConnectionPool` per database, `request.input(name, type, value)`, `WITH (NOLOCK)` for read-only analytics reads, request-level timeouts with `request.cancel()`.

**Q12. Two databases in one app?**
A `Database` class holding named pools (`primary`, `clinic`); repositories choose by name; `p-limit` caps concurrent operations (Appointment-CRM).

**Q13. What is Dynamic Data Masking?**
SQL Server hides column values in SELECT output for logins without `UNMASK` (e.g., `98XXXXXX28`), but WHERE predicates still compare the real values. My apps detect masked output, log the UNMASK requirement, and mask again in the UI for privacy.

**Q14. `NOLOCK` — pros and cons?**
No shared locks → no blocking on a busy OLTP table, but dirty reads are possible. Acceptable for a nightly analytics mirror, not for financial writes.

**Q15. T-SQL functions you converted to other dialects?**
`FORMAT(date,'yyyy-MM')` → `substr(date,1,7)`; `DATEDIFF(DAY,a,b)` → `julianday(b)-julianday(a)`; `ISNULL` → `COALESCE`; `CONVERT(VARCHAR(10), date, 23)` for ISO dates.

**Q16. Legacy password compatibility?**
Re-implemented ASP.NET `Rfc2898DeriveBytes` (PBKDF2, 1000 iterations, SHA1) → 32-byte key + 16-byte IV → AES-256-CBC over UTF-16LE text, base64 — so existing users log in unchanged.

---

## PostgreSQL

**Q17. `pg` Pool basics?**
`new Pool({...})`, `pool.query(text, values)`, handle `pool.on('error')` for idle clients, `SELECT NOW(), current_database()` ping before listen.

**Q18. Server-side cursors?**
`DECLARE cur CURSOR FOR …; FETCH 10000 FROM cur;` inside a transaction with `statement_timeout = 0` — used to build the Patient-CRM SQLite index from a 50M-row table without loading it into memory.

**Q19. `DISTINCT ON`?**
PostgreSQL-only: returns the first row per group according to ORDER BY. Perfect for "latest record per phone".

**Q20. Partial indexes?**
Index with a WHERE clause; small and fast for a specific filter (failed association reports; call-window filters).

**Q21. What was the data warehouse you read?**
The Ameyo dialer/ACD tables (~50M rows dialer calls, ~56M call-history legs); one call → many agent legs; `crt_object_id` groups legs of one interaction.

---

## MongoDB

**Q22. Schema design decisions?**
Message document stores normalised fields (`from`, `to`, `status`, `isRead`) plus `raw` payload for audit; Contact holds `lastBusinessNumberId` for routing; Email has a unique compound index `{address, project}`. Embed what is read together; index what is filtered/sorted.

**Q23. Compound indexes and order?**
Order matters: equality fields first, then sort field. `{to:1, isRead:1}` for unread counts; `{direction:1, timestamp:-1}` for chat history; `{owner:1, createdAt:-1}` for project lists.

**Q24. Aggregation pipeline examples?**
`$match → $group` for status/type breakdowns; `$dateToString {format:'%Y-%m-%d'}` + `$group` + `$sort` for daily series; `$lookup` to Contact for names in top contacts; zero-fill missing buckets in code.

**Q25. Upsert and idempotency?**
`findOneAndUpdate({sid}, {...}, {upsert:true})` so webhook retries do not duplicate messages.

**Q26. `insertMany({ordered:false})` — why?**
Continue inserting after a duplicate-key error; combined with the unique index gives DB-level dedup.

**Q27. Gotcha: aggregation and ObjectId?**
`find()` auto-casts string ids; `aggregate()` does not — you must pass `new ObjectId(id)` in `$match`. (Known issue in the Email-Extract MVP backlog.)

**Q28. Atlas connection issues?**
SRV DNS lookups can fail on some networks; I added a fallback that sets public DNS servers and retries.

---

## SQLite (as a read-model)

**Q29. Why SQLite in a server app?**
Local, zero-network, full SQL, single file, easy to rebuild. Ideal as a read-model/cache for reports.

**Q30. WAL mode?**
Write-Ahead Log lets readers continue while one writer writes. Set `journal_mode=WAL`, `synchronous=NORMAL`.

**Q31. Limitations and how you handled them?**
One writer at a time and synchronous API (better-sqlite3) → single Node process, worker concurrency 1, range guards, worker thread for index builds, atomic temp-file swap for full rebuilds.

**Q32. Schema migrations?**
Guarded `ALTER TABLE ADD COLUMN` in try/catch and a `pragma_table_info` probe to detect old shapes and rebuild cache tables; a `SCHEMA_VERSION` constant.

---

## Redis

**Q33. What did you use Redis for?**
Cache (QMS reports with versioned keys, Patient-CRM L2 search cache), OTP storage with TTL (ViewReports), dialer flow session (CRM), BullMQ broker (Email-Extract, ViewReports worker).

**Q34. Cache invalidation strategies you used?**
TTL; versioned key prefixes; key includes data stamp (`last_sync_at`); cache only successful non-empty results.

**Q35. What if Redis is down?**
All Redis calls are wrapped: errors logged, fall through to the next tier; the app keeps working slower. BullMQ needs Redis ≥5 — the API boots with queues disabled and logs the fix if the version is too old.

---

## Data engineering / ETL

**Q36. Describe your sync engine.**
Source registry → per-source cursor → chunked pull → sanitise → upsert in transaction → progress row → cancel flag between chunks → stale-job reconciliation on boot; inline cron or BullMQ worker.

**Q37. How do you validate migrated/derived data?**
Compare against known outputs (Cohorts vs Python: avg LTV 3846.91, total customers 1,366,125), spot-check with business users, log row counts per source (`source_stats`).

**Q38. Handling case sensitivity across engines?**
SQL Server default collation is case-insensitive; SQLite is case-sensitive by default → `COLLATE NOCASE` on the mirrored columns.

**Q39. Time zones?**
Cron in `Asia/Kolkata`; dates normalised to `YYYY-MM-DD` strings during mirroring; durations stored as `HH:MM:SS` strings in the dialer table converted in SQL.

---

## Rapid-fire

- **Normalisation vs denormalisation?** OLTP normalised (CRM tables); reporting denormalised (dialer warehouse, SQLite read-model).
- **ACID?** Atomic, Consistent, Isolated, Durable — SQLite transactions for cache writes.
- **N+1 problem?** Fetch related rows in one query (joins / `$lookup` / `$in`) — e.g., Patient-CRM builds all phone variants and queries once.
- **Soft delete?** `isArchived`, `isActive` flags (Email-Extract).
- **Connection pooling?** Reuse connections; pool sizes per DB; monitor pool stats every 60 s (CRM).
- **Backups?** Read-only mirrors are rebuildable; source DBs are backed up by DBA; frozen backup table merged in Cohorts.
