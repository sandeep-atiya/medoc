# Behavioral Questions — STAR answers from your real work

STAR = Situation → Task → Action → Result. Keep each answer to 60–90 seconds.

---

## 1. "Tell me about a time you solved a difficult technical problem."

**S:** In ViewReports, a single heavy report could freeze the whole API because better-sqlite3 is synchronous and IIS ran one Node process; the app pool would recycle mid-request.
**T:** Keep reports fast without losing the SQLite read-model benefits.
**A:** Added a `reportRangeGuard` middleware to cap date ranges, moved the multi-minute index build to a worker thread, moved long syncs to a separate BullMQ worker process, and keyed the in-process cache on the last-sync timestamp.
**R:** No more freezes or recycles; most reports return in milliseconds; sync and reporting no longer compete.

## 2. "Tell me about a time you found the requirement was wrong."

**S:** CallDropAutoDial spec said to read dropped calls from `acd_call_details` every 30 minutes.
**T:** Deliver a working job.
**A:** During testing every window returned zero rows. I investigated the warehouse and found that table is filled by a one-hour ETL, so a 30-minute window is always empty. I switched to the live parent table, verified with real data, and documented the correction next to the SQL and told operations.
**R:** The job produced correct call-back lists from day one; the doc prevented the same confusion later.

## 3. "Tell me about a production issue you handled."

**S:** After a QMS deploy, a feature flag (archive recordings on/off) appeared to be ignored in production.
**T:** Find why config changes did not apply.
**A:** Read iisnode logs, found the old process still running because `watchedFiles` did not include `src`. Fixed the list, added a startup log of every flag value, documented the `Restart-WebAppPool` command in `web.config`.
**R:** Deploys now restart correctly and can be verified from the log alone.

## 4. "Tell me about improving performance."

**S:** QMS patient enrichment used an official SQL Server view that took ~92 seconds.
**T:** Make the report usable for auditors.
**A:** Analysed the view (per-row OUTER APPLY), queried base tables with UNION ALL, wrote an idempotent index script for production, added per-query timeouts with cancellation so the page never hangs.
**R:** 200–350 ms; auditors got a one-screen report.

## 5. "Tell me about working with non-technical stakeholders."

**S:** QA auditors needed "everything about a call" but could not say which of the two call legs each field should come from.
**T:** Turn a vague ask into a precise spec.
**A:** Sat with them, wrote a 400-line data dictionary with field → source table → rule, listed assumptions to confirm (NTB rule, hangup-by leg), built the report, reviewed with real calls.
**R:** Report accepted; the assumptions list became the acceptance checklist.

## 6. "Tell me about a time you had to learn something quickly."

**S:** The WhatsApp inbox was my first real-time app; Exotel's webhook format and WhatsApp media rules were new.
**T:** Deliver a working inbox for two business numbers.
**A:** Set up ngrok to capture real payloads, saved them as Postman fixtures, learned Socket.IO event design, discovered WhatsApp rejected voice notes and solved it by transcoding via Cloudinary.
**R:** Live inbox with real-time updates and correct multi-number routing, plus 17 guides for the team.

## 7. "Tell me about a mistake you made."

**S:** Early on, I committed environment files with credentials into internal repos to move fast.
**T:** Keep delivery speed but fix hygiene.
**A:** Introduced `.env.example` files, environment-specific env files, envalid validation, and a plan to move secrets to server environment variables/vault; I now flag it in code reviews.
**R:** Newer projects have example files and validation; I can explain the risk and the fix clearly.

## 8. "Tell me about prioritising under pressure."

**S:** Appointment-CRM went live while ViewReports and QMS were in development; agents reported WhatsApp and consultation-form errors on the live server.
**T:** Fix production without derailing the other projects.
**A:** Time-boxed the live fix first (logs → env loading and error handling), shipped, verified with agents, then returned to the roadmap.
**R:** Fix committed the same day ("live server error logs issue resolved"); other deliveries stayed on schedule.

## 9. "Tell me about designing for safety."

**S:** CallDropAutoDial pushes real customers into a dialer — a bug would auto-call people repeatedly.
**T:** Go live safely.
**A:** DRY_RUN default, re-entrancy guard, single IIS process, two-stage dedup, reconnect-exclusion SQL, preview endpoint for operations, per-record logging, and a no-retry policy agreed with seniors.
**R:** Clean go-live; operations could verify the list before turning DRY_RUN off.

## 10. "Tell me about reusing work / improving your own process."

**S:** Eight apps in nine months alone.
**T:** Build faster without lowering quality.
**A:** Standardised the layered Express template, copied it (Patient-CRM → QMS explicitly), wrote a DDL-to-model generator, kept deployment runbooks.
**R:** Later apps took days instead of weeks; deployments became repeatable.

## 11. "Disagreement with a senior/manager?"

**S:** I proposed email alerts on Ameyo upload failures; seniors preferred log-only to avoid alert noise.
**A:** I implemented log-only but made the logs actionable (full failing payload, first 10 rejects) and added a manual trigger endpoint so operations could re-run.
**R:** Both goals met; documented the policy in code so the decision is visible.

## 12. "How do you handle ambiguity?"

Give the QMS assumptions example: build with a clearly named constant, flag it "confirm with business", list it in the README, and verify with real data before go-live.

## 13. "How do you ensure quality when you are the only developer?"

Consistent structure, validation at the edge, health endpoints, Postman/smoke scripts, parity checklists, UAT with real users, and comments explaining why decisions were made.

## 14. "Tell me about a time you said no / pushed back."

Example: refused to retry Ameyo uploads automatically because a retry could double-dial customers; proposed the preview + manual re-run instead.

## 15. "What motivates you?"

Seeing agents and managers use something daily that removed manual work — e.g., auditors no longer cross-checking three systems; managers running reports themselves.
