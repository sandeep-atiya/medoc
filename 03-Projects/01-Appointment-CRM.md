# Project 1 — Appointment-CRM (Call-Center CRM, Legacy ASP.NET → Node + React)

**Repo:** `D:\Appointment-CRM` (GitHub: Appointment-CRM)
**Period:** December 2025 → September 2026 (171 commits; most active Jan–Feb 2026; live fixes Aug–Sep 2026)
**Size:** ~934 source files, ~78,000 lines (backend ~49k, frontend ~29k)
**My role:** Sole full-stack developer — analysis of the legacy app, architecture, backend, frontend, integrations, IIS deployment, production support.

---

## 1. Pitches

**One line:**
> A call-center CRM used by agents inside the Ameyo dialer to book and manage clinic appointments, replacing a legacy ASP.NET WebForms application with a Node.js + React system on the same SQL Server databases.

**30-second pitch:**
> "The company had an old ASP.NET WebForms CRM that agents used inside the Ameyo dialer to book clinic appointments. I rewrote it as a React 19 frontend and a Node/Express 5 backend on top of the existing two SQL Server databases, keeping full parity with the legacy behaviour. It has 47 API modules — appointments, patients, dispositions, complaint tickets, consultation forms, courier tracking — plus integrations with the Ameyo dialer API, Exotel WhatsApp and an SMS gateway. I deployed it on Windows IIS with iisnode and it is used daily by the agents."

**2-minute pitch (add these points):**
- The legacy app had 460+ tables across two databases. I wrote a script that reads the SQL DDL and generates a model class for every table (463 models), and another script that scaffolds controllers and routes from those models. That saved weeks.
- Agents open the CRM as an iframe inside Ameyo. I built a stateless SSO flow: Ameyo passes user/session/campaign/extension via URL or postMessage; the backend resolves the agent from headers and caches it for 5 minutes.
- The UI has 23 disposition panels (Booking, Follow-up, Callback, Hold, Non-contact, Denied, Feedback, etc.) and a consultation module with 16 disease-specific JSON form schemas, draft autosave and "repeat consultation" prefill.
- On appointment save, the backend can auto-dispose the call in Ameyo, send a WhatsApp template through Exotel (with retry on 5xx/429 only) and an SMS through ValueFirst, and log the delivery status from the Exotel webhook.
- Two SQL Server connection pools (CRM DB + clinic DB) behind one `Database` class with a concurrency limit; Redis holds the dialer "flow session" to mirror ASP.NET session behaviour.
- Deployed on IIS + iisnode with 4 Node processes, health/readiness endpoints, Winston daily-rotate logs, and a production checklist.

---

## 2. Business problem

- Agents took patient calls in the Ameyo dialer and used an old WebForms page (a single 250 KB code-behind file) to book/reschedule appointments, record dispositions, and raise complaints.
- The old app was slow, hard to change, and tied to .NET Framework 4.5.
- Management wanted a modern, faster UI and an API-based backend that could later be reused by other tools (reports, WhatsApp, QMS) — without changing the database, because many other systems depend on it.

## 3. What I built — features

**Agent worklist (appointments-list):** Today, Hold, Follow-up, Confirm-Date-Later, Callback, Non-contact, DD follow-up (fresh/pending), consultations (direct / hot follow-up / verification), broadcast message, feedback, booking + assign, denied/cancel per city, clear leads, search by patient / appointment / mobile / conversion report.

**Appointments:** create, reschedule, change clinic, audit trail of changes, clinic address & pincode lookup, disease/problem master.

**Patients:** create/edit patient with full address block, patient history, call history by patient ID / mobile / merged patient ID.

**Dispositions & campaigns:** campaign master, campaign → disposition → sub-disposition mapping, dialer auto-dispose on save.

**Consultation module:** 16 disease-specific dynamic forms (diabetes, skin, hair, joint pain, heart, stomach, weight loss, etc.) rendered from JSON schemas; hooks for prefill, history, repeat consultation and draft autosave.

**Complaint tickets:** master + child tickets with category, priority, status, refund amount, new order/refund IDs.

**Orders & courier tracking:** order details, invoices, courier status parsers for Delhivery, XpressBees, ShipDelight (+ Shiprocket token models).

**Communication:** WhatsApp appointment confirmation template (9 parameters: name, date, time, ref id, clinic address, doctor, clinic contact, support number, map link) via Exotel; SMS via ValueFirst; delivery-status webhook stored in a log table.

**Admin:** users CRUD (admin only), role change, attendance, login history, page-rights based RBAC (OnlyView / CanDownload / CanEdit), IT-issue tickets, agent notes, statistics dashboard + today's call report (Recharts).

**Platform:** `/health`, `/health/status`, `/health/ready`, Swagger UI at `/api/docs` from an OpenAPI YAML.

## 4. Architecture

```
Ameyo Dialer (agent desktop)
   └─ iframe → React SPA (IIS static site, web.config SPA rewrite)
                 │ axios + x-user-id / x-session-id headers (Ameyo SSO), JWT for admin
                 ▼
          Express 5 API (IIS + iisnode, 4 node processes)
          routes → Joi validation → controllers (47) → services (45) → repositories (48)
                 │                                   │
                 │                                   ├─ dialer.service → Ameyo REST (dial/dispose/hangup)
                 │                                   ├─ whatsapp.service → Exotel Messages v2 (+ retry)
                 │                                   ├─ sms.service → ValueFirst
                 │                                   └─ courierTracking.service → Delhivery/XpressBees/ShipDelight
                 ▼
     Database class with 2 named pools (mssql/tedious):
        primary = CRM DB (353 tables)     clinic = Clinic DB (110 tables)
     Redis (cloud) = dialer flow session state
```

**Frontend structure:** two UI shells in one SPA — `pages/*` (dashboard + 23 panels) and `app-calling/*` (in-call agent screen: call history, appointments, notes, new customer, new appointment, order history, consultation). 7 Redux slices (auth, appointments, patients, agents, ui, dialer, disposition) with redux-persist for auth/ui; a FlowContext for the dialer flow.

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19.2, Vite (rolldown-vite 7), react-router-dom 6.30, Redux Toolkit 2 + react-redux 9 + redux-persist 6, Tailwind CSS 3.4, Recharts 3, react-select, react-toastify, FontAwesome + lucide + react-icons, axios |
| Backend | Node.js (ESM), Express 5.2, Joi 17 + express-joi-validation, jsonwebtoken + bcryptjs, helmet, cors, hpp, xss-clean, express-rate-limit, Winston + winston-daily-rotate-file, morgan, swagger-ui-express + yamljs, nodemailer, axios, dayjs, p-limit, node-cron, uuid |
| Database | Microsoft SQL Server via `mssql` 12 + `tedious` 19, raw parameterised SQL, two pools; Redis via ioredis |
| Tooling | ESLint, Prettier, Jest + Supertest, nodemon, cross-env |
| Deploy | IIS + iisnode (`web.config`), PM2 `ecosystem.config.js` (cluster mode) as alternative, `.env.development/.staging/.production` |

## 6. Database design notes

- No ORM. Every table has an **auto-generated model class** describing `database`, `schema`, `tableName`, `columns` (SQL type, nullable, identity, JS type), `primaryKey`, `foreignKeys`. Header: "AUTO-GENERATED MODEL — DO NOT EDIT MANUALLY". Loaded dynamically at runtime.
- Key tables: `tblAppointment` (app_id, pid, phone, clinic, app_date, app_time, rem_date, app_status, disposition_code, campaignId…), `Appointment` (richer version with AssignedBy/To, Modified audit columns), `tblPatient` (multiple phones, full address, problem), `AdminAuthentication` (user + page-rights CSV columns OnlyView/CanDownload/CanEdit, machine/IP/MAC tracking), `ComplaintTicketMaster`, `WhatsAppExotelLog` (sid, template, params, http code, delivery status, delivered on), `AppointmentAuditTrail`.
- RBAC tables: `PageRights`, `ParentNodePageRight`, `UserType`, `UserRole`, `webpages_Roles`.
- Campaign/disposition tables: `CampaignMaster`, `CampaignDispositionMapping`, `AddDisposition`, sub-dispositions.
- Courier tables: `CourierManagement`, `CourierStatus`, `CourierTrackingLinkMaster`, NDR tables.

## 7. Integrations (how each works)

| Integration | How |
|---|---|
| **Ameyo dialer** | Backend calls Ameyo REST commands: manual dial, dispose, dispose-and-dial, hangup, diagnostics. Auto-dispose on appointment save is configurable by env (which dispositions trigger it). |
| **Ameyo iframe SSO** | Frontend `AutoLoginHandler` reads `userId / sessionId / campaignId / extension` from URL query or `postMessage`, stores them; every API call sends `x-user-id` + `x-session-id`; a global middleware resolves username → numeric agent ID with a 5-minute in-memory cache. Redis stores the per-agent "flow" (campaign, extension) to mimic ASP.NET Session. |
| **Exotel WhatsApp** | `POST /v2/accounts/{sid}/messages` with HTTP Basic auth, template `appointment_template`, 9 body params. Retries only on 5xx / 429 / network errors (never on 4xx), max 3 attempts with base delay 250 ms. Never throws into the appointment flow — failures are logged in `WhatsAppExotelLog`. Delivery receipts arrive at a public webhook guarded by a secret query token. |
| **ValueFirst SMS** | HTTP POST with username/password/to/from/text; returns "Y"/"N" for parity with the legacy `SMS.cs`. |
| **Courier tracking** | Service with per-courier response parsers (Delhivery, XpressBees, ShipDelight; Ekart two-call flow). |
| **Email** | nodemailer helper with Gmail SMTP (ready, few call sites). |

## 8. Deployment (what you actually did)

- Backend `web.config`: iisnode handler on `src/server.js`, rewrite-all rule, `nodeProcessCountPerApplication="4"`, `maxConcurrentRequestsPerProcess="4096"`, logging to `logs/iisnode`, `node_env=production`, WebSocket enabled, `httpErrors PassThrough`.
- Frontend `public/web.config` (copied into `dist/`): SPA rewrite to `/index.html`, `no-cache` for `index.html`, 365-day cache for `/assets`, `X-Content-Type-Options: nosniff`.
- App pool: No Managed Code, **AlwaysRunning**, Idle Time-out 0, Queue length 5000, Preload enabled, Application Initialization (documented in `IISNODE_PRODUCTION_CHECKLIST.md`).
- Node server tuned for IIS: `keepAliveTimeout`, `headersTimeout`, `requestTimeout`; a 60-second DB pool-stats log loop.
- Alternative: PM2 `ecosystem.config.js` (`crm-api`, cluster mode, `instances: max`, `max_memory_restart 500M`).
- Builds: `vite build --mode production`; backend `cross-env NODE_ENV=production node src/server.js`.

## 9. Challenges and how I solved them (use as STAR stories)

1. **460+ tables, no time to hand-write models**
   *Action:* wrote `generate-models-from-sql.js` to parse the SQL DDL export and emit one model per table, plus `scaffold-controllers-routes-from-models.js`. *Result:* 463 models generated, consistent naming, zero typos in column names, new tables supported by re-running the script.

2. **Keeping parity with a 250 KB legacy code-behind file**
   *Action:* created `PARITY_CHECKLIST.md`, mirrored legacy endpoints under `/legacy/*`, kept the same return values (e.g., SMS returns "Y"/"N"), commented each service with the legacy C# source it replaces. *Result:* agents saw identical behaviour; old integrations kept working.

3. **Agents work inside an iframe — no normal login screen**
   *Action:* built header-based SSO from the Ameyo iframe + a `ProtectedRoute` that polls session storage for up to 2 seconds before showing "Unauthorized" (because postMessage arrives after first render). *Result:* zero-click login for agents.

4. **WhatsApp failures must never block a booking**
   *Action:* WhatsApp service never throws; retries only on transient errors; every attempt logged with HTTP code, error code, delivery status; webhook updates delivery status later. *Result:* booking flow always completes; support can see exactly why a message failed.

5. **Two databases in one app**
   *Action:* one `Database` class holding named pools (`primary`, `clinic`), repositories choose the pool by name, `p-limit` caps in-flight DB operations. *Result:* no pool exhaustion under agent load.

6. **Live server error logs for WhatsApp and consultation form (Sep 2026)**
   *Action:* traced through iisnode logs, fixed env loading and error handling. *Result:* commit "WhatsApp and Consultation form live server error logs issue has been resolved".

## 10. Results / impact

- Legacy WebForms CRM replaced by a modern SPA + API used daily by call-center agents.
- 47 API modules / 50 route files / ~80 endpoints documented in OpenAPI.
- Appointment confirmations sent automatically on WhatsApp + SMS with delivery tracking.
- Reusable API layer later consumed by other internal tools.

## 11. Interview questions specific to this project (with answers)

**Q: Why Express 5 and not NestJS?**
A: The team is small and the legacy system is procedural; Express 5 with a strict folder pattern gave the structure of a framework without the learning curve, and Express 5 has native async error handling.

**Q: Why no ORM with SQL Server?**
A: The schema is legacy with 460+ tables, odd column names and stored logic; raw parameterised SQL keeps full control over performance and matches the legacy queries one-to-one. The generated model classes give me column metadata without ORM overhead.

**Q: How do you prevent SQL injection?**
A: Every query uses `mssql` `Request.input()` parameters; Joi validates input shape before it reaches the service; helmet/hpp/xss-clean on top.

**Q: How does authentication work?**
A: Two paths. Admin/user-management routes use JWT (`authenticate` + `authorize(roles)` + `requireAdmin`). Agent routes use stateless header SSO from the Ameyo iframe (`x-user-id`, `x-session-id`) resolved by a global middleware with a 5-minute cache. The WhatsApp status webhook is public but guarded by a secret token.

**Q: How do you handle the Ameyo call lifecycle?**
A: Ameyo opens the CRM with the call context; when the agent saves a disposition, the backend can call Ameyo's dispose API (configurable list of dispositions) and, if needed, hang up or dial the next number.

**Q: What does redux-persist persist and why?**
A: Only `auth` and `ui` slices, so a page refresh inside the iframe does not log the agent out or reset layout, but appointment/patient data is always fresh from the API.

**Q: How are the consultation forms built?**
A: Each disease has a JSON schema (sections → fields with type, options, validation). A generic `ConsultationFormEditor` renders any schema; `useDraftAutosave` stores drafts; `useRepeatConsultation` prefills from the last consultation.

**Q: How do you handle retries for WhatsApp?**
A: Retry only when the error is transient (5xx, 429, network). 4xx means our payload is wrong, so retrying is pointless. Max 3 attempts with a base delay of 250 ms.

**Q: How did you test it?**
A: Postman collections per module, Swagger UI for contract checks, Jest + Supertest for health, and agent UAT inside the real Ameyo iframe on a staging IIS site before production.

**Q: How do you deploy an update?**
A: Build frontend → copy `dist` to the IIS site folder; copy backend `src` + `npm ci --omit=dev`; iisnode restarts on watched-file change or I recycle the app pool; verify `/health/ready` and one real booking.

## 12. Be careful — what NOT to claim

- `socket.io`, `bull`, `agenda`, `multer`, `express-session` are in package.json but **not used**. Do not say the CRM has real-time sockets, job queues or file uploads.
- The `node-cron` schedulers are **stubs** and are never started. Do not claim scheduled jobs in this project (claim them in CallDropAutoDial / Cohorts / ViewReports instead).
- The rate limiter is intentionally set to unlimited for the agents' shared office IP. If asked, explain: "per-agent keys via x-user-id are on the roadmap".
- Most agent feature routes rely on the Ameyo header SSO, not JWT. Say "JWT for admin routes, header SSO for agent routes".
- Secrets exist in committed env files. If asked about secret management, say the improvement is a secret store / CI-injected env.

## 13. What I would improve next

- Enable per-agent rate limiting keyed by `x-user-id`.
- Move WhatsApp/SMS sending to a BullMQ queue for retries outside the request.
- Add Socket.IO for live worklist refresh instead of polling.
- Add unit tests for services and contract tests from the OpenAPI file.
- Move secrets out of the repo (Windows environment variables / secret manager).

## 14. Resume bullets

- Re-engineered a legacy ASP.NET WebForms call-center CRM into a React 19 + Node.js/Express 5 application on the existing SQL Server databases (2 connection pools, 460+ tables), preserving full functional parity for daily agent use.
- Built a SQL-DDL-to-code generator producing 463 model classes and controller/route scaffolds, cutting weeks of boilerplate; delivered 47 API modules with Joi validation, OpenAPI docs, RBAC and audit trails.
- Integrated the Ameyo dialer (dial/dispose/hangup, iframe SSO), Exotel WhatsApp Business API (templated confirmations with retry/backoff and delivery-status webhooks), ValueFirst SMS and courier tracking APIs.
- Deployed on Windows Server IIS with iisnode (multi-process, AlwaysRunning app pool, health/readiness probes, rotating logs) and supported the system in production.

## 15. Keywords for ATS

React 19, Redux Toolkit, redux-persist, React Router, Tailwind CSS, Recharts, Vite, Node.js, Express 5, REST API, Joi, JWT, RBAC, SQL Server, T-SQL, mssql, tedious, Redis, Winston, Swagger/OpenAPI, Helmet, IIS, iisnode, PM2, Ameyo dialer, Exotel WhatsApp API, SMS gateway, webhooks, legacy migration, ASP.NET WebForms, code generation
