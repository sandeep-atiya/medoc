# Security & Authentication Interview Questions

Be honest: some internal tools have no auth by design (they run inside an authenticated host on a LAN). Show that you understand the risks and have the roadmap.

---

## Authentication

**Q1. Which authentication methods have you implemented?**
1. **JWT + bcrypt** (Appointment-CRM admin routes, Email-Extract).
2. **JWT access + refresh tokens** (15 min / 7 d, separate secrets, refresh stored on user, revocable) — Email-Extract.
3. **Email OTP two-factor** — ViewReports (OTP in Redis with TTL) and Cohorts (stateless HMAC-in-JWT OTP).
4. **Stateless header SSO** from the Ameyo dialer iframe — Appointment-CRM.
5. **API key header** for machine-to-machine — WhatsApp Dashboard.
6. **Legacy-compatible password check** (ASP.NET PBKDF2 + AES-256-CBC) — ViewReports, Cohorts.

**Q2. How does your OTP flow work without storing the OTP?**
Login validates the password, generates a 6-digit OTP, emails it, and returns a 5-minute `tempToken` JWT that contains an HMAC-SHA256 of the OTP keyed by the server secret. Verify recomputes the HMAC and compares with `crypto.timingSafeEqual`. No DB write, cannot be brute-forced offline, expires automatically.

**Q3. Where do you store JWTs on the client and why?**
localStorage/sessionStorage with an axios interceptor. Risk: XSS. Mitigations: helmet CSP, no `dangerouslySetInnerHTML`, short expiry, refresh tokens. HttpOnly cookies are safer against XSS but harder inside third-party iframes (cookies blocked), which is why the embedded apps use headers.

**Q4. JWT vs session?**
JWT is stateless (good for multiple processes and iframes); sessions need a store. I used JWT everywhere, Redis only for OTP/flow state, and no express-session.

**Q5. How do you hash passwords?**
bcrypt with 12 salt rounds; never log or return the password (`select:false` in Mongoose).

---

## Authorization

**Q6. How did you implement RBAC?**
- ViewReports: comma-separated permission strings on the user row → `canViewReport(key)` / `canDownloadReport(key)` middleware per route + `requireSuperAdmin`; the same key list filters the sidebar.
- Appointment-CRM: `authorize(...roles)`, `requireAdmin` on user management, DB page-rights (OnlyView/CanDownload/CanEdit).
- Email-Extract: `authorize(roles)` + `adminOnly` factory; ownership-scoped queries (`owner: userId`) to prevent IDOR.
- WhatsApp Dashboard: role gate on report pages (front-end) — backend improvement pending.

**Q7. What is IDOR and how do you prevent it?**
Insecure Direct Object Reference — accessing another user's record by changing an id. Prevent by scoping every query by the authenticated user/owner, which I do in Email-Extract (`Project.findOne({_id, owner: userId})`).

---

## Input & transport security

**Q8. How do you prevent SQL injection / NoSQL injection?**
Parameterised SQL everywhere; Joi validation of shape/length; `express-mongo-sanitize` in the CRM; never build SQL from string concatenation of user input.

**Q9. XSS?**
React escapes by default; helmet sets CSP/nosniff; no raw HTML injection; validation strips unexpected characters (email cleaner).

**Q10. CSRF?**
APIs use bearer tokens/API keys in headers (not cookies), so classic CSRF does not apply; webhooks are POST-only with secret tokens where applicable.

**Q11. Rate limiting and brute force?**
express-rate-limit tiers: auth 10/15 min, global 200/15 min, scrape 30/hour; strict limiter on the manual job trigger; OTP expiry 5 minutes.

**Q12. Helmet — what does it set?**
Security headers: CSP, `X-Content-Type-Options`, `Referrer-Policy`, HSTS (disabled on plain-HTTP LAN), frame options (removed for embeddable tools with explicit `frame-ancestors`).

**Q13. CORS policy?**
Allowlist of known origins; `*` only for the WhatsApp Socket.IO in the first version (improvement noted).

**Q14. HTTPS?**
Cloudways app runs on HTTPS via Let's Encrypt; LAN IIS apps run HTTP internally with an `HTTPS_ENABLED` flag that switches HSTS/upgrade-insecure on when a certificate is bound.

---

## Integration security

**Q15. How do you secure webhooks?**
CRM WhatsApp status webhook: public path guarded by a secret query token and explicitly whitelisted in the auth middleware. WhatsApp Dashboard webhooks: currently open (Exotel does not sign) → improvement: shared secret header + IP allowlist + idempotent upsert (already done).

**Q16. Open proxy risk in the recording stream — how did you avoid it?**
Upstream URL is rebuilt from a whitelisted base + regex-validated ID; client cannot supply a URL; timeouts; abort on disconnect.

**Q17. Third-party credentials?**
Kept in env files, loaded at boot, never sent to the browser (except the WhatsApp UI API key — flagged as an improvement).

---

## Privacy / data protection

**Q18. Patient data — what did you do?**
Phone masking in the UI (`98XXXXXX28`, `******4248`), awareness of SQL Server Dynamic Data Masking and UNMASK, read-only access to clinical tables, no PHI in logs where possible, embedded tools sit behind the authenticated portal on the LAN. Roadmap: audit logging of who viewed which patient/call, HTTPS everywhere, JWT on all embedded tools.

**Q19. Logging sensitive data?**
Winston logs contain request ids and error codes; I avoid logging tokens; improvement: redact phone numbers and payloads in production logs (the WhatsApp app currently logs webhook bodies — noted).

---

## Secrets & code hygiene (honest answers)

**Q20. Are secrets in your repos?**
"Yes, in some internal repos env files were committed for speed. I know this is wrong for anything public. My improvement plan: gitignore all env files, keep only `.env.example`, inject via server environment/vault, rotate keys."

**Q21. Dependency hygiene?**
Some package.json files list unused libraries from scaffolds. I would run `depcheck`, remove dead deps, and pin versions. Express 5 forced me to drop `xss-clean`/`hpp` (they mutate `req.query`).

---

## Rapid-fire

- **Salt?** Random value added before hashing so identical passwords hash differently (bcrypt does it internally).
- **HMAC?** Keyed hash for integrity/authentication — used for OTP tokens.
- **Constant-time compare?** Prevents timing attacks — `crypto.timingSafeEqual`.
- **Least privilege?** Read-only DB logins for reporting apps; only the needed tables.
- **Token expiry?** 8 h (ViewReports), 15 min access / 7 d refresh (Email-Extract), 5 min OTP temp token.
- **Security headers on static SPA?** nosniff, cache rules, frame-ancestors per host.
