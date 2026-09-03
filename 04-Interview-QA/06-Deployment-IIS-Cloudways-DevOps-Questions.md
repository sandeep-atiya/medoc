# Deployment / DevOps Interview Questions — IIS + iisnode, Cloudways + PM2, environments

You deployed 6 apps on Windows IIS with iisnode and 1 on Cloudways (Linux) with PM2. Read `07-Deployment-Playbook` for the step-by-step; this file is the Q&A.

---

## IIS + iisnode

**Q1. What is iisnode and why use it?**
An IIS module that hosts Node.js processes behind IIS: IIS handles ports/SSL/process management and forwards requests to Node over a named pipe. Used because the company standard is Windows Server + IIS and the ops team already manages IIS sites.

**Q2. Walk me through deploying a Node API on IIS.**
1. Install Node, iisnode, URL Rewrite (and ARR if reverse-proxying).
2. Create a site (or application) pointing to the backend folder; dedicated app pool: **No Managed Code**, Start Mode **AlwaysRunning**, Idle Time-out **0**, Regular recycling **0**, Preload enabled (Application Initialization feature).
3. `web.config`: `<handlers><add name="iisnode" path="src/server.js" verb="*" modules="iisnode"/>`, a rewrite rule sending all requests to `src/server.js`, `<iisnode>` settings (process count, logging, `node_env`, `watchedFiles`, `nodeProcessCommandLine`), `httpErrors existingResponse="PassThrough"`, WebSocket enabled.
4. `npm ci --omit=dev`; `.env.production` in place.
5. Browse `/health` → verify; check `logs/iisnode` on error.

**Q3. What is special about `PORT` under iisnode?**
iisnode sets `process.env.PORT` to a named pipe like `\\.\pipe\...`, not a number. Never `parseInt` it; listen on whatever value is given.

**Q4. Why did some apps run 1 Node process and others 4?**
`nodeProcessCountPerApplication`: SQLite-backed apps (ViewReports) and the cron job (CallDropAutoDial) must be **1** — multiple writers corrupt SQLite and multiple processes would double-fire the cron / double-push to Ameyo. Stateless APIs (Appointment-CRM) run 4 for throughput.

**Q5. What is `watchedFiles` and what bug did it cause?**
iisnode restarts Node when a watched file changes. The default did not include `src\*`, so uploading new code kept the old process running and env toggles seemed ignored. Fixed by listing `src\**\*.js`, `.env.*`, `web.config`; documented `Restart-WebAppPool` as the recovery.

**Q6. How do you host the React build on IIS?**
Static site pointing to `dist/`; `web.config` with a URL Rewrite rule: if the request is not a file/folder, rewrite to `/index.html`; MIME maps for `.json/.woff2/.svg`; `clientCache` 365 days for `/assets`, no-cache for `index.html`; `X-Content-Type-Options: nosniff`; for embedded tools remove `X-Frame-Options` and set `frame-ancestors`.

**Q7. How did you avoid CORS between the SPA and API on IIS?**
Option A: separate sites + CORS allowlist in Express. Option B (Cohorts): one site, ARR + URL Rewrite proxy `/api/*` → `http://localhost:<nodePort>/api/*` — same origin, no CORS.

**Q8. Where are the logs?**
`logs/iisnode/*.txt` (stdout/stderr per process), Winston `logs/app.log`/`error.log` (rotating), IIS logs; plus `/health/status` endpoint.

**Q9. How do you handle the working directory difference under iisnode?**
`process.cwd()` is not the project root; resolve `.env` and data files from `import.meta.url` / `__dirname`.

**Q10. How do you set environment variables on IIS?**
`.env.<NODE_ENV>` files loaded by dotenv (iisnode has no `--env-file`), `node_env="production"` in `web.config`; envalid fails fast if anything is missing.

**Q11. Rate limiting behind IIS?**
Enable `enableXFF` in iisnode and `app.set('trust proxy', 1)` so the real client IP is seen; custom keyGenerator fallback when IP is undefined.

**Q12. How do you keep a background job alive on IIS?**
AlwaysRunning + Idle Time-out 0 + Preload + no periodic recycle; or run it outside IIS as a Windows Task Scheduler task (`runSyncOnce`) or an NSSM service (`worker.js`).

**Q13. How would you add HTTPS?**
IIS binding with a certificate (internal CA or Let's Encrypt via win-acme); then enable HSTS/upgrade-insecure in helmet (currently switched off via `HTTPS_ENABLED` flag because the LAN apps run on HTTP).

---

## Cloudways + PM2 + Apache

**Q14. Walk me through the Cloudways deployment.**
1. Create the application on Cloudways (server on DigitalOcean/other), SSH in.
2. Clone the repo into `applications/<app>/public_html`.
3. Backend: `npm ci --omit=dev`, create `.env` from the production example, `pm2 start ecosystem.config.cjs`, `pm2 save`, `pm2 startup`.
4. Frontend: `npm run build`; Apache serves `dist`.
5. `.htaccess` at the root: `RewriteCond %{REQUEST_FILENAME} !-f` → `RewriteRule ^(.*)$ http://127.0.0.1:5000/$1 [P,L]`, plus a rule for `Upgrade: websocket` → `ws://127.0.0.1:5000`; SPA rewrite to `index.html` skipping `/api`; CORS headers incl. `X-API-Key`; `mod_deflate`, `mod_expires`.
6. Map the domain in Cloudways, enable Let's Encrypt SSL, point the Exotel webhook to the domain.
7. Run `check-deployment.sh` / `test-deployment.sh`.

**Q15. PM2 fork vs cluster?**
Fork = one process (used for WhatsApp because Socket.IO without a Redis adapter needs a single instance). Cluster = N processes sharing the port (CRM ecosystem file uses `instances: 'max'`). Cluster needs stateless apps or sticky sessions + Redis adapter for sockets.

**Q16. Important PM2 settings you used?**
`max_memory_restart: '500M'`, `autorestart`, `max_restarts: 10`, `min_uptime: '10s'`, log file paths, env blocks per environment, `pm2 logs`, `pm2 restart <name>`, `pm2 save` + `pm2 startup` for reboot persistence.

**Q17. Why `[P,L]` in .htaccess and what module is needed?**
`P` = proxy the request (needs `mod_proxy`, `mod_proxy_http`, `mod_proxy_wstunnel` for WebSockets); `L` = last rule.

**Q18. How did you test webhooks locally?**
ngrok tunnel to the local port; Exotel webhook temporarily pointed to the ngrok URL; Postman collections to replay real payloads.

---

## Environments, config, secrets

**Q19. How do you manage multiple environments?**
`.env.development / .env.staging / .env.production` for backend (dotenv + envalid) and `VITE_*` env files per Vite mode for the frontend; build with `vite build --mode staging|production`; `/health/status` reports which files loaded.

**Q20. Secrets — what is the current state and what would you improve?**
Honest answer: env files are in the repo for these internal apps. Improvement: `.gitignore` all env files, keep `.env.example` only, inject secrets via server environment variables or a vault, rotate keys, and never ship API keys in the frontend bundle.

**Q21. Zero-downtime deploys?**
Not fully today. On IIS: overlapping app-pool recycle helps; on PM2 cluster: `pm2 reload` does rolling restarts. Plan: build artifacts in CI, deploy to a new folder, switch the site path.

**Q22. CI/CD?**
Currently manual with scripts (`deploy.sh`, runbooks). Plan: GitHub Actions → build frontend + install backend → deploy via WinRM/SSH → run health checks.

**Q23. Docker?**
Not used for these deployments (Windows/IIS standard). I know the model (Dockerfile, compose for API + worker + Redis + Mongo) and would use it for the Email-Extract stack.

---

## Monitoring & troubleshooting

**Q24. A deploy is done but users see old behaviour — steps?**
Check `/health/status` build/flags → check iisnode process restart (watchedFiles) → recycle app pool → clear browser cache (index.html no-cache should handle) → check env file loaded.

**Q25. API returns 502/503 behind IIS — steps?**
iisnode logs for crash → `/health/ready` for DB status → app-pool state → port/pipe conflicts → node memory (`max-old-space-size`).

**Q26. High memory / crashes during sync?**
Chunk size, cursors, single writer, `max_memory_restart`, move sync to a separate worker process.

**Q27. How do you verify a release?**
Health endpoints, one real transaction (booking, message, report), log tail, and a rollback folder ready.

---

## Rapid-fire

- **Reverse proxy?** A server that forwards client requests to another server (IIS ARR, Apache mod_proxy).
- **Named pipe?** OS-level IPC channel used by iisnode instead of a TCP port.
- **App pool?** IIS worker process boundary; settings control lifetime and recycling.
- **Sticky sessions?** Route a client to the same process; needed for Socket.IO in cluster mode without an adapter.
- **Blue-green?** Two environments; switch traffic after verifying the new one.
- **Health vs readiness?** Health = process alive; readiness = dependencies OK.
