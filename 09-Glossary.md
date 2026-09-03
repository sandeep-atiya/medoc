# Glossary — simple meanings of terms used in these documents

| Term | Simple meaning | Where you used it |
|---|---|---|
| ACD | Automatic Call Distributor — the dialer part that routes inbound calls to queues/agents | PostgreSQL warehouse tables |
| Ameyo | The call-center dialer software the company uses; has REST APIs and stores call data | CRM, QMS, Patient-CRM, CallDropAutoDial, ViewReports |
| API key | A secret string sent in a header to identify a trusted client | WhatsApp Dashboard |
| ARR | IIS Application Request Routing — lets IIS act as a reverse proxy | Cohorts |
| Atomic file swap | Write to a temp file then rename, so readers never see a half-written file | Patient-CRM SQLite build |
| Backoff (exponential) | Wait longer between each retry (250 ms, 500 ms, 1 s…) | WhatsApp, CRM, Email-Extract |
| bcrypt | Slow password hashing with salt | CRM, Email-Extract |
| BullMQ | Redis-based job queue library for Node | Email-Extract, ViewReports worker |
| Cheerio | Fast HTML parser (jQuery-like) for scraping without a browser | Email-Extract |
| Cohort | A group of customers who first bought in the same month | Cohorts-Analytics |
| Compound index | Index on multiple fields, order matters | MongoDB models, SQLite |
| CORS | Browser rule that blocks cross-origin API responses unless allowed by headers | All SPAs |
| Cursor (server-side) | Database keeps the result and gives it in batches | Patient-CRM ETL |
| DDL | SQL that defines tables (CREATE TABLE…) | Model generator |
| DISTINCT ON | PostgreSQL: keep first row per group | CallDropAutoDial |
| DLR | Delivery Report — provider tells you sent/delivered/read/failed | WhatsApp, CRM |
| DNIS | The number a customer dialed (used to attribute TV channels) | ViewReports |
| Disposition | The outcome code an agent sets after a call | CRM, QMS |
| DRY_RUN | Run everything but do not perform the real side-effect | CallDropAutoDial |
| Dynamic Data Masking | SQL Server hides column values from logins without UNMASK | Patient-CRM, QMS |
| envalid | Library that validates environment variables at startup | Several backends |
| ETL | Extract → Transform → Load data from one store to another | ViewReports, Cohorts, Patient-CRM |
| Exotel | Provider used for WhatsApp Business messaging (and SMS in some cases) | WhatsApp Dashboard, CRM |
| Graceful shutdown | On stop signal, finish requests, close DB pools, then exit | Most backends |
| Helmet | Express middleware that sets security headers | Most backends |
| HMAC | Keyed hash used to verify data was not changed | Cohorts OTP |
| Idempotent | Doing it twice has the same effect as once | Webhook upserts, dedup |
| IDOR | Accessing someone else's record by changing an id | Prevented by owner-scoped queries |
| iisnode | IIS module that runs Node.js apps | 6 apps |
| Joi | Request validation library | Most backends |
| JWT | Signed token carrying user identity; stateless auth | CRM, ViewReports, Cohorts, Email-Extract |
| Keyset pagination | Page by last key (`id > last`) instead of OFFSET | Cohorts |
| Leg (call leg) | One segment of a call; a transferred call has two legs | QMS, ViewReports |
| LRU cache | Keeps the most recently used items, evicts the oldest | Patient-CRM, ViewReports |
| LTV | Lifetime value — total revenue from a customer | Cohorts |
| Mongoose | MongoDB object modelling for Node | WhatsApp, Email-Extract |
| MX record | DNS record that says which mail server accepts email for a domain | Email validation |
| Named pipe | OS communication channel used by iisnode instead of a TCP port | IIS apps |
| NTB | New-to-Brand — first-time customer | QMS, Cohorts, ViewReports |
| OTP | One-time password sent by email for two-factor login | ViewReports, Cohorts |
| Parity | New system behaves exactly like the old one | CRM migration |
| Partial index | Index with a WHERE clause — small and fast for one filter | ViewReports, CallDrop |
| PM2 | Process manager for Node on Linux (restart, logs, cluster) | WhatsApp, CRM config |
| Puppeteer | Headless Chrome automation | Email-Extract |
| RBAC | Role/permission based access control | CRM, ViewReports |
| Range request | HTTP header asking for part of a file; enables audio seeking | QMS proxy |
| Read-model | A copy of data shaped for fast reads (reports) | ViewReports, Cohorts, Patient-CRM |
| Re-entrancy guard | A flag that prevents the same job from running while it is already running | CallDropAutoDial |
| Redis | In-memory store for cache, OTP, queues | Several |
| Reverse proxy | Server that forwards requests to another server | ARR, Apache |
| SerpAPI | Service that returns Google search results as JSON | Email-Extract |
| Socket.IO | Library for real-time events over WebSockets | WhatsApp Dashboard |
| SPA | Single Page Application — one HTML file, client-side routing | All frontends |
| SSO (header) | Trusting identity passed by the host app via headers | CRM in Ameyo iframe |
| TanStack Query | Server-state caching library for React | ViewReports, QMS, Email-Extract |
| Tedious | Node driver for SQL Server (used by `mssql`) | CRM |
| Timing-safe compare | Compares strings in constant time to stop timing attacks | OTP verification |
| UNION ALL | Combine rows from two queries without removing duplicates | QMS, Cohorts |
| Upsert | Update if exists, insert if not | Webhooks, sync engine |
| Vite | Frontend build tool/dev server | All frontends |
| WAL (SQLite) | Write-Ahead Log mode — readers don't block the writer | SQLite apps |
| watchedFiles (iisnode) | Files whose change restarts the Node process | IIS apps |
| Webhook | The provider calls your URL when something happens | WhatsApp, CRM |
| Winston | Logging library with transports and rotation | Most backends |
| Worker thread | Runs JavaScript on another thread for CPU-heavy work | ViewReports index build |
| Zustand | Small React state library | ViewReports |
