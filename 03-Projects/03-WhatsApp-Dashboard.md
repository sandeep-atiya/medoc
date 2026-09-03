# Project 3 — WhatsApp Dashboard (Real-Time WhatsApp Business Inbox, MERN + Socket.IO)

**Repo:** `D:\whatsapp-dashboard-mern` (GitHub: whatsapp-dashboard-mern)
**Period:** November/December 2025 → February 2026 (22 commits) — your first Node + React production app
**Size:** ~65 source files, ~7,100 lines + 17 documentation guides + Postman collections
**My role:** Sole full-stack developer — Exotel integration, webhook design, MongoDB schema, Socket.IO real-time, chat UI, reports, Cloudways deployment.

---

## 1. Pitches

**One line:**
> A real-time WhatsApp Business inbox where agents chat with customers through the Exotel WhatsApp API, with automatic multi-business-number routing, unread tracking, media via Cloudinary and an admin reporting area — deployed on Cloudways with PM2.

**30-second pitch:**
> "The company talks to customers on two WhatsApp Business numbers through Exotel. I built a MERN application: Exotel sends inbound messages and delivery receipts to my Express webhook, I store them in MongoDB Atlas and push them to agents' browsers with Socket.IO. Agents reply from the chat UI, and the backend automatically sends the reply from the same business number the customer wrote to. It has unread badges, media upload through Cloudinary with audio transcoding for WhatsApp, and reports built with MongoDB aggregation. It is deployed on Cloudways with PM2 behind an Apache reverse proxy that also proxies WebSockets."

**2-minute pitch (add):**
- Message lifecycle: outbound saved as `queued` → Exotel accepts → DLR webhook updates `sent/delivered/read/failed` by message SID → socket event to the UI.
- Retry with exponential backoff on Exotel calls (3 attempts, 250 ms base; only on network/5xx/429).
- MongoDB compound indexes for chat lists and unread counts (`{to, isRead}`, `{from, to, isRead}`, `{direction, timestamp}`).
- Reports: totals, status and type breakdown, daily inbound/outbound series (`$dateToString`), top contacts joined to contact names, plus paginated unified reports.
- Development with ngrok so Exotel could reach the local webhook; Postman collections and 17 markdown guides documented for the team.

---

## 2. Business problem

- Customer conversations on WhatsApp were handled on phones; managers had no shared inbox, no history, no reporting.
- Two business numbers (two brands) had to be handled in one screen without mixing them up.
- Agents needed instant notification of new messages and unread counts.

## 3. What I built — features

- **Inbound webhook** (`POST /api/webhooks/exotel` and root `POST /`) parsing Exotel's `{ whatsapp: { messages: [...] } }` payload; handles `incoming_message` and `dlr` callback types; unknown types stored for audit.
- **Send message API** with Joi validation (E.164 phone regex), text + media, template category support.
- **Automatic business-number routing:** inbound resolves which business number the customer wrote to and stores it on the message and the contact; outbound reuses the contact's last business number, then a default, then the first configured number.
- **Unread system:** `isRead`/`readAt`, unread counts per contact for "my number", mark-read endpoint, `message:unread` socket event, badge component.
- **Real-time:** Socket.IO server (websocket + polling transports, ping tuning); events `connection:success`, `contact:upsert`, `message:inbound`, `message:outbound`, `message:unread`; client context with auto-reconnect and a connection status indicator.
- **Media:** multer → Cloudinary; audio (`ogg/mp3/m4a/aac/amr/opus`) uploaded as video resource and transcoded to MP3 so WhatsApp accepts it; image compression in the browser before upload.
- **Contacts:** list, by id, by phone, create, update, delete, bulk import.
- **Reports (Admin/Developer roles):** aggregate dashboard, contacts/messages/media reports, unified report, single-message detail.
- **Chat UI:** sidebar with contact list and unread badges, chat window with bubbles, media bubbles, date separators, emoji picker, drag-and-drop attachments, socket status.
- **Privacy:** phone masking utility in the UI.

## 4. Architecture

```
Customer WhatsApp ⇄ Exotel WhatsApp Business API
                        │ webhooks (incoming_message, dlr)          ▲ send / mark-read (Basic auth)
                        ▼                                           │
   Apache (.htaccess reverse proxy: / → 127.0.0.1:5000, ws:// upgrade) on Cloudways
                        ▼
   Express 4 API (PM2 fork mode)  ── x-api-key middleware for /api/*
      controllers: webhooks, messages (send/history/unread/report), contacts, media
      services/exotel: http client + request wrapper with retry/backoff + payload builders
      Socket.IO server (app.set('io'))  ──►  broadcasts to all connected agents
                        ▼
   MongoDB Atlas (Mongoose 7): Message, Contact, Media         Cloudinary (media storage)
                        ▲
   React 19 SPA (Vite 7, Tailwind 3, socket.io-client, axios with x-api-key)
```

## 5. Exact tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 7 (SWC), react-router-dom 7, Tailwind CSS 3.4, socket.io-client, emoji-picker-react, react-dropzone, react-player, browser-image-compression, react-icons / remixicon, axios |
| Backend | Node.js ESM, Express 4.18, Socket.IO 4.8, Mongoose 7, Joi 17, multer 2, cloudinary 2, axios, express-async-errors, morgan, uuid (request IDs), dotenv |
| Data | MongoDB Atlas (with DNS-SRV fallback to public resolvers on connection failure) |
| Deploy | Cloudways (Linux) — PM2 `ecosystem.config.cjs`, Apache `.htaccess` ×3 (proxy, SPA rewrite, CORS, gzip, caching), `deploy.sh`, `check-deployment.sh`, `test-deployment.sh`, ngrok for local webhooks |

## 6. Database design (Mongoose)

- **Message:** `sid`, `direction` (inbound/outbound), `from`, `to`, `businessNumberId`, `callbackType`, `messageType`, `status`, `exoStatusCode`, `exoDetailedStatus`, `description`, `timestamp`, `customData`, `templateCategory`, `content` (object), `raw` (original payload), `isRead`, `readAt`, timestamps. Indexes on sid/direction/from/to/businessNumberId/status/timestamp/isRead + compound `{direction, timestamp:-1}`, `{to, isRead}`, `{from, to, isRead}`.
- **Contact:** `phone` (unique), `waId`, `profileName`, `lastMessageAt`, `lastSeenAt`, `lastBusinessNumberId`.
- **Media:** `provider` (cloudinary/exotel), `url`, `mimeType`, `sizeBytes`, `publicId`, `resourceType`, `originalFilename`, `meta`.

## 7. Integrations

| Integration | How |
|---|---|
| Exotel WhatsApp v2 | `POST /v2/accounts/{sid}/messages` to send; `PUT` to mark read; template endpoints per WABA; Basic auth with key/token; response normalised to `AppError` with upstream body on failure. |
| Exotel webhooks | Inbound messages and DLRs; both public endpoints; message matched by `sid`. |
| Cloudinary | Upload from multer temp file; audio forced to `resource_type: video` + MP3 transcode. |
| MongoDB Atlas | SRV connection with fallback DNS servers. |

## 8. Deployment (Cloudways)

1. Cloudways app → SSH → clone repo into `applications/<app>/public_html`.
2. Backend: `npm ci --omit=dev`, copy `.env.production.example` → `.env`, `pm2 start ecosystem.config.cjs` (app `whatsapp-backend`, fork mode, `max_memory_restart 500M`, autorestart, logs in `logs/`).
3. Frontend: `npm run build` → `dist` served by Apache; `.htaccess` SPA rewrite to `index.html`, skip `/api`.
4. Root `.htaccess`: `RewriteRule` proxy all non-file requests to `http://127.0.0.1:5000` and WebSocket upgrade to `ws://127.0.0.1:5000`; CORS headers including `X-API-Key`; mod_deflate + mod_expires.
5. Domain mapped in Cloudways, SSL via Let's Encrypt in the panel.
6. Exotel webhook URL pointed to the production domain; verified with `test-deployment.sh` and Postman.

## 9. Challenges and solutions

1. **Two business numbers in one inbox.** → Store `businessNumberId` on every message and `lastBusinessNumberId` on the contact; outbound picks the right `from` automatically. Result: replies always come from the number the customer used.
2. **Webhooks unreachable during development.** → ngrok tunnel to local server; multiple webhook aliases (root `POST /`, `/api/webhooks/exotel`, legacy path) so Exotel config changes never broke delivery.
3. **WhatsApp rejected voice notes.** → Upload audio to Cloudinary as video resource and transcode to MP3, which WhatsApp accepts.
4. **Delivery status arrived before the outbound save finished.** → Upsert by `sid` in the DLR handler so order of arrival does not matter.
5. **WebSockets through Apache on shared hosting.** → `.htaccess` proxy rule for the `Upgrade: websocket` header; socket client allowed both websocket and polling transports.
6. **Transient Exotel errors.** → Retry wrapper with exponential backoff only for network/5xx/429.

## 10. Results / impact

- Shared, searchable WhatsApp inbox for the whole team with live updates.
- Zero mis-sent replies from the wrong business number.
- Reporting on message volume, delivery status and top contacts for management.
- Documented (17 guides + Postman) so the ops team could self-serve.

## 11. Interview questions specific to this project

**Q: Why MongoDB here and SQL Server elsewhere?**
A: Messages are document-shaped with varying payloads (`content`, `raw`), high write rate, simple access patterns (by contact, by time). Mongo with compound indexes fits well; the CRM data stays in SQL Server where it belongs.

**Q: How does Socket.IO scale with PM2?**
A: Currently one process in fork mode, so a single Socket.IO instance is fine. To scale to cluster mode I would add the Redis adapter so events broadcast across processes and enable sticky sessions.

**Q: How do you secure the API?**
A: Machine-to-machine `x-api-key` middleware on `/api/*`; webhooks are public by nature (Exotel calls them). Improvement: verify a shared secret/signature on webhooks and move UI login to JWT.

**Q: How do you avoid duplicate messages from webhook retries?**
A: Upsert by Exotel `sid`; the message document is idempotent on that key.

**Q: How are unread counts computed efficiently?**
A: Aggregation grouped by `from` for `to = myNumber AND isRead = false`, backed by the `{to, isRead}` and `{from, to, isRead}` compound indexes.

**Q: What happens if Exotel is down?**
A: Send returns a 502 `exotel_error` with the upstream body after 3 retries; message stays `queued`/`failed` and the UI shows it; nothing is lost because the message document was saved first.

## 12. Be careful — what NOT to claim

- UI login is **hardcoded client-side credentials** with roles in localStorage — not JWT. Say: "API key auth for the API; simple role-based UI login; JWT planned."
- Template management endpoints return **501 Not Implemented**; do not claim template CRUD is live (sending template-category messages works).
- Redux Toolkit is installed but **not used**; state is `useState` + Context.
- No charting library; reports are HTML tables/cards.
- No webhook signature verification; no rate limiter mounted. Present both as next steps.
- Backend README describes a "70 APIs scaffold" that does not match the code; do not quote it.

## 13. What I would improve next

- JWT login with server-side roles; webhook signature verification; mount the rate limiter.
- Redis adapter for Socket.IO + PM2 cluster.
- Template management UI (endpoints exist in the Exotel service layer).
- Split the 1,000-line messages controller into services.

## 14. Resume bullets

- Built a real-time WhatsApp Business inbox (MERN + Socket.IO) integrated with the Exotel WhatsApp API: inbound/DLR webhooks, idempotent message upserts, automatic multi-business-number routing, unread tracking with compound indexes, Cloudinary media with audio transcoding, and MongoDB aggregation reports.
- Deployed on Cloudways with PM2 and an Apache reverse proxy (including WebSocket upgrade), with deployment/check/test scripts and Postman collections; documented in 17 operational guides.

## 15. Keywords

MERN, React 19, Socket.IO, WebSockets, Node.js, Express, MongoDB Atlas, Mongoose, aggregation pipeline, compound indexes, Exotel, WhatsApp Business API, webhooks, DLR, Cloudinary, multer, Joi, retry/backoff, PM2, Apache, .htaccess, Cloudways, ngrok, Tailwind
