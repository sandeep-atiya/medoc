# Tricky / Trap Questions — and honest answers that still sound strong

Interviewers use these to test honesty and depth. Never bluff; give the real state + the improvement.

---

**Q1. "Did you really build all of this alone? It's a lot for one person."**
"Yes. Two things made it possible: the same layered template reused across projects, and code generation for the 463 models. Also, several apps are small and focused — QMS, Patient-CRM and CallDropAutoDial are 2–3k lines each. The big ones are the CRM (~78k lines) and ViewReports (~59k). I can walk through any file."

**Q2. "Why are there many small projects instead of one big product?"**
"Each solved a specific business problem for a different team and was deployed separately on IIS as its own site. They share the same structure and databases, and some are embedded in others (QMS and Patient-CRM inside the ViewReports portal, the CRM inside the dialer)."

**Q3. "Your package.json lists Socket.IO / Bull / Sequelize in a project that doesn't use them. Why?"**
"Some projects were scaffolded from a full-stack template, and unused dependencies were left in. It's a hygiene issue I'd fix with `depcheck`. I only claim what is used: Socket.IO is live in the WhatsApp app; BullMQ in Email-Extract and the ViewReports worker; there is no Sequelize anywhere — I use raw SQL."

**Q4. "Some apps have no authentication. Isn't that a security problem?"**
"They are internal LAN tools embedded inside an authenticated host (ViewReports portal with JWT + OTP, or the Ameyo dialer). That boundary is not enforced inside QMS/Patient-CRM themselves, which I've listed as the next step: share the JWT middleware. The public-facing app (WhatsApp) uses an API key and runs on HTTPS."

**Q5. "Credentials are committed in your repos."**
"True for some internal repos — a speed trade-off I would not repeat. The fix: gitignore env files, keep `.env.example`, inject secrets from the server environment or a vault, rotate keys."

**Q6. "You used raw SQL — isn't an ORM better?"**
"For legacy schemas with 460+ tables and heavy reporting queries, raw parameterised SQL gave me control and performance (e.g., DISTINCT ON, cursors, temp tables). I used Mongoose where the data model fits documents. I'm comfortable with ORMs too; it's a per-project decision."

**Q7. "Why Windows IIS and not Docker/Kubernetes/cloud?"**
"Company infrastructure standard — Windows Server managed by the IT team, internal LAN, on-prem databases. I learned iisnode deeply (named pipes, watched files, process counts). For the public WhatsApp app I used Cloudways with PM2. I'd be happy to use Docker/CI in a new environment."

**Q8. "No unit tests?"**
"Limited automated tests: Jest + Supertest in the CRM, smoke scripts and Postman collections elsewhere, plus UAT with real users. As a solo developer under delivery pressure I invested in validation and health checks first. With a team I would add service-level unit tests and contract tests from the OpenAPI spec."

**Q9. "Why SQLite in a server application? That's unusual."**
"As a read-model, not the system of record. Full SQL at local speed, single file, rebuildable. I handled the single-writer limit with one process and a worker lock. At larger scale I'd move it to PostgreSQL/ClickHouse."

**Q10. "What's a limitation of your ViewReports design?"**
"One writer, one process, synchronous SQLite. Mitigated with range guards, caching and a separate worker, but horizontal scaling needs a different read-model."

**Q11. "Your WhatsApp UI login is hardcoded."**
"Yes — v1 for a small internal team, with API-key protection on the API. JWT login with server-side roles is the planned next step."

**Q12. "How do you know your cohort numbers are correct?"**
"I validated against the original Python output on known figures (avg LTV 3846.91, 1,366,125 customers) and fixed collation and date-format differences until they matched."

**Q13. "Why did the Email-Extract project stop at MVP?"**
"It was a self-initiated prototype to build a queue-based scraping architecture. It has known issues in the backlog (ObjectId cast in aggregations, React Query v5 hook change, export button not wired). I present it as a learning/prototype project."

**Q14. "The GitHub account/commits are under a different name."**
"The repositories are hosted under a company/team GitHub account, and the commit identity is the machine's configured git user. I can walk through any part of the code live to show ownership." *(Adjust this to the truth of your situation before the interview.)*

**Q15. "What would you not do again?"**
"Committing env files; leaving scaffold dependencies; putting a 1,000-line controller in the WhatsApp app instead of services; and starting without TypeScript."

**Q16. "Which project has the weakest code and why?"**
"WhatsApp Dashboard — my first Node/React app: one large controller, routes re-mounted by array position, no rate limiter mounted. It works and is documented, but I'd refactor it into services and add proper auth."

**Q17. "How would you onboard a new developer to these repos?"**
"Start with the layered structure, run `test:connections`/health, read the README and runbooks, then take one 5-file module as a template."

**Q18. "How do you handle a request you think is a bad idea?"**
Give the Ameyo retry example: explain the risk (double-dialing), propose a safer alternative (preview + manual trigger), document the decision.

**Q19. "What happens if Redis goes down in QMS/Patient-CRM?"**
"Every cache read/write is wrapped; the app logs and falls through to the next tier. Slower, not broken."

**Q20. "Explain something you built that you are not fully sure was the right choice."**
"Storing OTP as an HMAC inside a temp JWT — elegant and stateless, but it means I can't invalidate an OTP early or count attempts without a store. In ViewReports I used Redis with TTL instead, which allows attempt limits."

---

## The honesty rule

If you do not know: "I haven't used that, but here is the closest thing I've done…" — then map it to a real example. Interviewers value that more than a wrong confident answer.
