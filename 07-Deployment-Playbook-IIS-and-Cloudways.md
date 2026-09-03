# Deployment Playbook — How You Deploy (IIS + iisnode, and Cloudways + PM2 + Apache)

This is exactly what you did, written as a repeatable procedure. Learn it so you can explain each step and *why*.

---

## Part A — Windows Server: IIS + iisnode (used for 6 apps)

### A1. One-time server setup
1. Install **Node.js LTS** (≥18) for all users.
2. Install **iisnode** (x64) — adds the `iisnode` module to IIS.
3. Install **IIS URL Rewrite** module.
4. Install **Application Request Routing (ARR)** only if you want to reverse-proxy `/api` to Node from a static site (used in Cohorts). Enable proxy in ARR settings.
5. Enable the IIS feature **Application Initialization** (for Preload / AlwaysRunning).
6. Create a folder per app, e.g. `D:\Application\NodeJs\<AppName>\backend` and `\frontend`.

### A2. Backend site
1. **App pool:** New pool `<AppName>Pool` → .NET CLR: **No Managed Code** → Advanced: Start Mode **AlwaysRunning**, Idle Time-out **0**, Regular Time Interval (recycling) **0**, Rapid-Fail Protection **off**, Maximum Worker Processes **1** (must be 1 for SQLite-backed apps and cron jobs; can be more for stateless APIs, but Node process count is controlled by iisnode, not IIS).
2. **Site:** Add Website → name, physical path = backend folder, binding = HTTP on a dedicated port (or a host name), app pool = the one above. Preload Enabled = true.
3. **web.config** in the backend folder:

```xml
<configuration>
  <system.webServer>
    <handlers>
      <add name="iisnode" path="src/server.js" verb="*" modules="iisnode" />
    </handlers>
    <rewrite>
      <rules>
        <rule name="NodeApp">
          <match url="/*" />
          <action type="Rewrite" url="src/server.js" />
        </rule>
      </rules>
    </rewrite>
    <iisnode
      nodeProcessCountPerApplication="1"           <!-- 1 for SQLite/cron apps; 2–4 for stateless APIs -->
      nodeProcessCommandLine="node.exe --max-old-space-size=4096"
      maxConcurrentRequestsPerProcess="4096"
      loggingEnabled="true"
      logDirectory="logs\iisnode"
      devErrorsEnabled="false"
      node_env="production"
      enableXFF="true"
      watchedFiles="web.config;.env;.env.production;src\*.js;src\**\*.js" />
    <httpErrors existingResponse="PassThrough" />
    <webSocket enabled="true" />
    <security><requestFiltering><requestLimits maxAllowedContentLength="52428800" /></requestFiltering></security>
  </system.webServer>
</configuration>
```

4. Copy code, then `npm ci --omit=dev` in the backend folder.
5. Put `.env.production` in place. Your `config/env.js` loads `.env.${NODE_ENV}` via dotenv using a path from `import.meta.url` (iisnode's cwd is different) and validates with envalid.
6. In `server.js`: `const port = process.env.PORT || 5000;` — **do not parseInt** (iisnode passes a named pipe). `app.set('trust proxy', 1)`.
7. Browse `http://<host>:<port>/health` and `/health/ready`. Check `logs\iisnode\*.txt` if it fails.
8. Give the app-pool identity (or IIS_IUSRS) modify rights on `logs\`, `data\`, `cache\` folders.

### A3. Frontend (React/Vite) site
1. `npm ci && npm run build -- --mode production` (uses `.env.production` → `VITE_API_URL`).
2. Create a site pointing to the `dist` folder (or copy `dist` to the site path). Static site, any app pool.
3. `public/web.config` (copied into `dist` by Vite):

```xml
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="SPA" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
            <add input="{REQUEST_URI}" pattern="^/api/" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>
      </rules>
    </rewrite>
    <staticContent>
      <remove fileExtension=".json" /><mimeMap fileExtension=".json" mimeType="application/json" />
      <remove fileExtension=".woff2" /><mimeMap fileExtension=".woff2" mimeType="font/woff2" />
      <clientCache cacheControlMode="UseMaxAge" cacheControlMaxAge="365.00:00:00" />
    </staticContent>
    <httpProtocol>
      <customHeaders>
        <add name="X-Content-Type-Options" value="nosniff" />
        <!-- for tools embedded in another app: -->
        <remove name="X-Frame-Options" />
        <add name="Content-Security-Policy" value="frame-ancestors *" />
      </customHeaders>
    </httpProtocol>
  </system.webServer>
</configuration>
```
   Add a location rule so `index.html` gets `Cache-Control: no-cache` while `/assets` is cached for a year.

### A4. Same-origin API with ARR (Cohorts pattern)
In the **frontend** site's `web.config`, before the SPA rule:
```xml
<rule name="ApiProxy" stopProcessing="true">
  <match url="^api/(.*)" />
  <action type="Rewrite" url="http://localhost:<backendPort>/api/{R:1}" />
</rule>
```
Requires ARR with proxy enabled. Result: SPA and API on one origin — no CORS.

### A5. Deploying an update (checklist)
1. Build frontend → replace `dist` (keep the previous folder as rollback).
2. Copy backend `src` (+ `package.json`), run `npm ci --omit=dev` if deps changed.
3. Confirm `.env.production` unchanged or updated.
4. iisnode restarts on watched files; if not: `Import-Module WebAdministration; Restart-WebAppPool -Name "<AppName>Pool"`.
5. Verify `/health/status` (env files loaded, feature flags) and `/health/ready`.
6. Do one real transaction (booking / report / message).
7. Tail `logs\iisnode` and Winston logs for 5 minutes.

### A6. Background jobs on Windows
- Inside the API process: node-cron + AlwaysRunning pool (CallDropAutoDial, Cohorts).
- Separate process: **Windows Task Scheduler** running `npm run sync:once:prod`, or an **NSSM** service running `node src/worker.js` (ViewReports).

### A7. Troubleshooting table
| Symptom | Cause | Fix |
|---|---|---|
| 500 / "iisnode encountered an error" | Node crashed at boot (env missing) | Read `logs\iisnode`; envalid message tells the variable |
| Old code still running | watchedFiles missing `src` | Fix `watchedFiles`; recycle pool |
| Port in use / nothing listens | `parseInt(PORT)` on named pipe | Use raw `process.env.PORT` |
| Cron ran twice / duplicate pushes | >1 Node process | `nodeProcessCountPerApplication="1"`, Max Worker Processes 1 |
| App dies at night | Idle timeout / recycling | Idle 0, recycle 0, AlwaysRunning, Preload |
| SPA deep links 404 | No rewrite | SPA rule to `/index.html` |
| Rate limiter error `ERR_ERL_UNDEFINED_IP_ADDRESS` | No client IP behind iisnode | `enableXFF`, `trust proxy`, custom keyGenerator |
| Report freezes API | Sync SQLite long query | Range guard, cache, worker process |

---

## Part B — Cloudways (Linux): PM2 + Apache reverse proxy (WhatsApp Dashboard)

### B1. Create & connect
1. Cloudways → Add Application (custom/PHP stack is fine; Node runs via PM2) → note the app folder `applications/<id>/public_html`.
2. Enable SSH/SFTP; connect as the master user.
3. Ensure Node LTS is available (Cloudways provides Node; otherwise use nvm) and `npm i -g pm2`.

### B2. Backend
```bash
cd ~/applications/<id>/public_html/backend
npm ci --omit=dev
cp .env.production.example .env   # then edit real values
pm2 start ecosystem.config.cjs     # app: whatsapp-backend, script src/index.js, fork mode, PORT 5000
pm2 save
pm2 startup                        # follow the printed command so PM2 survives reboots
pm2 logs whatsapp-backend
```
`ecosystem.config.cjs` keys: `name`, `script`, `cwd`, `instances: 1`, `exec_mode: 'fork'`, `autorestart`, `max_memory_restart: '500M'`, `max_restarts: 10`, `min_uptime: '10s'`, `error_file/out_file`, `env: { NODE_ENV: 'production', PORT: 5000 }`.

### B3. Frontend
```bash
cd ../frontend && npm ci && npm run build   # dist served by Apache
```

### B4. Apache `.htaccess` (root of public_html)
```apache
RewriteEngine On
# WebSocket upgrade → Node
RewriteCond %{HTTP:Upgrade} =websocket [NC]
RewriteRule ^(.*)$ ws://127.0.0.1:5000/$1 [P,L]
# API and non-file requests → Node
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ http://127.0.0.1:5000/$1 [P,L]
```
Frontend `.htaccess`: SPA rewrite to `index.html` except `/api`; CORS headers (`Access-Control-Allow-Headers: X-API-Key, Content-Type`); `mod_deflate`; `mod_expires` for assets.

### B5. Domain, SSL, webhooks
1. Cloudways → Domain Management → add domain; DNS A record to server IP.
2. SSL → Let's Encrypt → install (auto-renew).
3. Point Exotel webhook to `https://<domain>/api/webhooks/exotel`.
4. Run `check-deployment.sh` / `test-deployment.sh`; test with Postman; verify a Socket.IO connection from the browser (status indicator).

### B6. Operating
- `pm2 restart whatsapp-backend`, `pm2 reload` (cluster), `pm2 logs`, `pm2 monit`.
- Rollback: keep previous release folder; `pm2 restart` after switching.
- Scaling plan: `exec_mode: 'cluster'` + Socket.IO Redis adapter + sticky sessions.

---

## Part C — Local development conventions

- `npm run dev` (nodemon) with `.env.development`; frontend `vite` with `/api` proxy.
- `start.bat` / `stop.bat` (Cohorts) to launch both.
- `npm run test:connections` (QMS) to verify DB/Redis before deploy.
- `npm run job:once` (CallDropAutoDial) to run one cycle without cron; `DRY_RUN=true` locally.
- ngrok for webhook development.

---

## Part D — Environment file convention

```
.env.example        # committed, placeholders only (target state)
.env.development    # local
.env.staging        # staging site
.env.production     # live (should NOT be committed — improvement)
```
Frontend: `VITE_API_URL`, `VITE_SOCKET_URL`, `VITE_APP_NAME` per mode.
Backend: `NODE_ENV`, `PORT`, DB hosts/users/passwords, `REDIS_*`, `JWT_SECRET`, third-party keys, feature flags (`WHATSAPP_ENABLED`, `DRY_RUN`, `AMEYO_ARCHIVAL_ENABLED`, `HTTPS_ENABLED`), `CORS_ORIGINS`, `SYNC_*`, `CRON_SCHEDULE`, `TZ`.
Validated at boot with envalid; `/health/status` reports which files were loaded.
