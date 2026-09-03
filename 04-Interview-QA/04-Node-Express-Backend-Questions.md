# Node.js / Express Backend Interview Questions — answered with your projects

---

## Node fundamentals

**Q1. How does Node handle concurrency with one thread?**
Event loop + libuv thread pool: I/O (DB, HTTP, files) is asynchronous and non-blocking; callbacks/promises resume when I/O completes. CPU-heavy work blocks the loop. *Where it mattered:* better-sqlite3 is **synchronous**, so a heavy report query blocks the whole process — I added a date-range guard and moved index builds to a **worker thread** (ViewReports).

**Q2. What are worker threads and when did you use them?**
`worker_threads` run JS on another thread with its own event loop. I used one to build a partial SQLite index (minutes long) so the API kept responding.

**Q3. Event loop phases — why should a backend developer care?**
Timers → pending callbacks → poll (I/O) → check (`setImmediate`) → close. Long synchronous work in any phase delays everything; `process.nextTick` and microtasks run between phases. Care = never do sync CPU work in a request path.

**Q4. CommonJS vs ESM?**
All my backends use ESM (`"type": "module"`, `import/export`). Gotchas I hit: `__dirname` not available (use `import.meta.url`), CJS-only packages need `createRequire` (SerpAPI client), and iisnode's working directory differs so env files are resolved from `import.meta.url`.

**Q5. Streams — did you use them?**
Yes: piping an upstream audio response to the client with Range headers (QMS), ExcelJS streaming exports, PostgreSQL cursors (batch fetch) for large syncs.

**Q6. How do you handle uncaught errors?**
Central error middleware; `process.on('unhandledRejection'/'uncaughtException')` handlers that log and decide: exit for genuine bugs, continue for operational network errors (ECONNRESET, ETIMEDOUT, EPIPE) — because on IIS an exit killed in-flight syncs. Graceful shutdown on SIGINT/SIGTERM closes the HTTP server and DB pools with a 10 s forced-exit timer.

**Q7. Memory management — what did you tune?**
`--max-old-space-size=4096` in the iisnode command line for sync-heavy apps; PM2 `max_memory_restart 500M`; cursors/keyset pagination so big result sets never sit in memory; LRU caches with max keys.

---

## Express

**Q8. Middleware order in your apps?**
`helmet` → `cors` → `compression` → body parsers (limits) → request id → morgan/Winston → rate limiter → auth (global or per router) → guards (range guard, permissions) → routes → 404 → error handler. Public webhooks are mounted **before** the auth middleware on purpose.

**Q9. Express 5 differences that affected you?**
Async handlers propagate rejections automatically; `req.query` is a read-only getter (so middlewares that mutate it throw — I stopped using xss-clean/hpp there and wrote validated values to `req.callId`); path-to-regexp changes for wildcard routes.

**Q10. How do you structure a feature?**
`routes/x.routes.js` (HTTP only) → `validations/x.validation.js` (Joi) → `controllers/x.controller.js` (thin, wraps `ApiResponse`) → `services/x.service.js` (business logic, caching, orchestration) → `repositories/x.repository.js` (SQL). `asyncHandler` wraps controllers; `ApiError` carries status codes.

**Q11. Input validation — how?**
Joi schemas for body/query/params via a generic `validate(schema, source)` middleware; env validated at boot with envalid (fail fast with a readable error).

**Q12. Error handling pattern?**
`ApiError(statusCode, message, details)`; error middleware maps known errors (Mongoose ValidationError → 422, CastError → 400, duplicate key 11000 → 409, JWT errors → 401), hides stack in production, logs with request id. Reporting apps follow a "never-500 for missing data" contract: degrade to `null` + `warnings[]`.

**Q13. Rate limiting?**
express-rate-limit with tiers (global/auth/scrape), `trust proxy` behind IIS/Apache, custom `keyGenerator` fallback for iisnode where the IP can be undefined; sized for a shared office IP where needed.

**Q14. CORS?**
Allowlist of origins (and any port for office hosts), credentials when needed; or avoid CORS entirely by reverse-proxying `/api` on the same origin (IIS ARR, Apache `.htaccess`).

**Q15. File uploads?**
multer with memory storage (10 MB, xlsx TV schedule parsed by ExcelJS in ViewReports) and disk storage to a temp folder then Cloudinary upload (WhatsApp).

**Q16. API documentation?**
OpenAPI YAML served with swagger-ui-express at `/api/docs` (Appointment-CRM); Postman collections elsewhere.

**Q17. Health checks?**
`/health` (liveness), `/health/ready` (pings SQL Server, PostgreSQL, Redis → 200 or 503), `/health/status` (which env files loaded, feature flags). IIS/monitoring can hit these.

---

## Auth & sessions (backend view)

**Q18. JWT flow you implemented?**
Login → verify password (bcrypt, or legacy ASP.NET AES scheme) → (optional OTP step) → sign JWT with expiry → client sends `Authorization: Bearer` → `authenticate` middleware verifies and attaches user → `authorize(roles)` / `canViewReport(key)` check permissions.

**Q19. Access + refresh tokens?**
15-minute access token, 7-day refresh token with separate secrets; refresh stored on the user document so it can be revoked (Email-Extract).

**Q20. Why stateless header SSO in the CRM?**
The app runs inside the dialer iframe where cookies are unreliable; the dialer already authenticated the agent, so I trust its user/session ids, resolve them server-side, and cache the lookup for 5 minutes.

**Q21. How do you store passwords?**
bcrypt (12 rounds) for new systems; for the legacy user table I re-implemented the existing ASP.NET encryption (PBKDF2 1000 iterations → AES-256-CBC over UTF-16LE) so users did not need to reset passwords.

---

## Background jobs & queues

**Q22. node-cron vs BullMQ — when?**
node-cron: simple time-based jobs in one process (CallDropAutoDial every 30 min with timezone; Cohorts daily 2 AM). BullMQ: work that must be queued, retried, distributed, or run in another process (Email-Extract pipeline; ViewReports sync worker with repeatable job, concurrency 1, 10-minute lock).

**Q23. How do you prevent a cron from running twice?**
Single process (IIS worker count 1), an in-process `running` flag, and idempotent work (dedup, upsert by key).

**Q24. Retry strategy?**
Only transient errors; exponential backoff (`250 * 2^(attempt-1)`), capped attempts; log each attempt; for Ameyo uploads a deliberate no-retry policy to avoid double-dialing customers.

**Q25. BullMQ specifics you know?**
Queue + Worker + Redis (`maxRetriesPerRequest: null`), `attempts`, `backoff` (fixed/exponential), `removeOnComplete/Fail` caps, `concurrency`, repeatable jobs, job ids for cancellation.

---

## Real-time

**Q26. Socket.IO server setup?**
Attach to the HTTP server, `path: /socket.io`, transports websocket + polling, ping timeout/interval tuned, CORS config; store `io` on `app` so controllers can emit; events: `message:inbound`, `message:outbound`, `message:unread`, `contact:upsert`. Scaling plan: Redis adapter + sticky sessions.

---

## Logging & observability

**Q27. Logging setup?**
Winston with console + file transports, daily rotation (CRM) or size rotation (5–10 MB × 5 files), JSON in production, Morgan piped into Winston, exception/rejection handlers; request ids via uuid; startup logs print feature flags and useful URLs.

---

## Integrations (backend)

**Q28. Calling third-party APIs safely?**
axios with timeouts, `transformResponse` when the API returns JSON as `text/plain` (Ameyo), Basic auth headers from env, whitelisted base URLs (never from client), `https.Agent` options where certificates are internal, and payload builders separated from HTTP code.

**Q29. Webhook handling?**
Public route mounted before auth; validate shape; idempotent upsert by provider id; store raw payload; respond 200 quickly; then broadcast/enqueue. Improvement: signature/secret verification (done in the CRM with a secret token).

---

## Testing

**Q30. How do you test backends?**
Postman collections, Jest + Supertest for endpoints (CRM), smoke scripts (`test:connections`, `job:once`, `smoke_test`), health endpoints in staging, and manual UAT. Next: unit tests per service with mocked repositories.

---

## Rapid-fire

- **`process.env.PORT` gotcha on iisnode?** It is a named pipe string, not a number — never `parseInt` it.
- **Graceful shutdown?** Stop accepting connections, close server, close pools, force-exit timer with `.unref()`.
- **Compression?** `compression` middleware for JSON; static assets compressed by IIS/Apache.
- **Security headers?** helmet; disable HSTS/upgrade-insecure when serving plain HTTP on a LAN.
- **Body size limits?** JSON limits set; multer limits for uploads.
- **Idempotency?** Upsert by external id (message SID), unique indexes, dedup before push.
- **Config?** dotenv per environment + envalid schema; secrets should move to a vault.
