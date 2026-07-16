# Resenas — External Dependencies & Risks

Ordered by (likelihood × impact). "Owner" = the operator (business owner), since external
tasks can't be done by a coding session.

---

## R1 — Google Business Profile API approval (BLOCKING for Phase B)

**What:** The Business Profile APIs require a formal access application (GCP project +
business-justification form). Until approved, quota is **0 QPM** — the API is unusable.
Approval has historically taken from days to several weeks, with rejections for vague
justifications.

**Impact if late:** Phase B (auto-posting, review management — the paid software function)
cannot start. Phase A is unaffected.

**Mitigation:**
- Submit the application **in week 1**, in parallel with PR-01 — do not wait for Phase A
  to finish. Owner task; needs the production GCP project and a clear justification
  ("agency software managing client GBP profiles with their consent" — be explicit about
  operator-approved, human-in-the-loop publishing).
- Plan already sequences all GBP-write work last (PR-18+), so the approval clock runs
  concurrently with ~all of the build.
- If rejected: respond/appeal with clarified use case. Fallback while re-applying:
  operator posts/replies manually via the GBP app using drafts from the admin panel
  (PR-21/23's drafting works without the publish leg — publishing is an isolated module).

## R2 — OAuth consent screen verification (`business.manage` scope)

**What:** Separate from R1. A production OAuth consent screen using the sensitive
`business.manage` scope needs Google's app verification (privacy policy URL, domain
verification, possibly a demo video). Unverified apps are capped at 100 test users and
show a scary warning screen — fatal for non-technical clients.

**Impact if late:** PR-19's onboarding flow scares clients off or blocks entirely.

**Mitigation:** Start verification as soon as the domain + a privacy policy page exist
(the policy page can ship inside PR-16). Test-user mode is sufficient for all development
and the first pilot clients (add their Googles as test users manually). Keep the scope
list minimal — exactly one sensitive scope.

## R3 — GBP profile suspensions

**What:** Google suspends business profiles routinely (edit bursts, category changes,
sometimes just API activity on a fresh connection). A suspended client profile makes our
posts/replies fail and the client will blame the tool.

**Mitigation:**
- Human-approved, low-frequency publishing only (already the design — D11); no bulk edits
  of profile fields in v1 scope at all (we only post and reply).
- Detect: BP API errors for suspended locations → set connection `status='error'`, notify
  operator (email), pause that connection's cron work automatically.
- Operator playbook note in `docs/OPERATOR-GUIDE.md` (PR-24): reinstatement request flow,
  and never promise clients immunity.

## R4 — Instagram public data access is unreliable

**What:** Unauthenticated Instagram profile fetches get login-walled, rate-limited by IP,
or the markup changes silently. Hostinger's shared egress IPs make blocking more likely.

**Mitigation:** Fully designed-in (PLAN D7): single fetch, short timeout, any failure →
`instagramDegraded`, score reweighting, neutral report copy, manual override fields in
admin. The IG area is capped at 15 % of the score precisely so its loss never invalidates
a report. **No scraping arms race** — if degradation becomes the norm, the manual field is
the product.

## R5 — Hostinger managed-Node constraints

**What:** Single process, no worker, request-duration limits, modest memory, ephemeral
disk on redeploys, cron only via hPanel-scheduled HTTP calls, shared egress IPs.

**Impact:** Long analyses or PDF rendering can hit timeouts/OOM; in-memory state dies on
restart; disk caches vanish.

**Mitigation:** The architecture is built around this (PLAN §1, D3, D6): step executor
with ≤20 s steps, all state in MySQL, pure-JS PDF (no Chromium), disk cache best-effort
only, cron = secret-gated routes. PR-01 documents exact deploy steps; PR-17 verifies
behavior under the real host before launch. Residual risk: if a real request limit
< 30 s surfaces, shrink step budgets (PSI step falls back to skip — already graceful).

## R6 — Places API cost creep / quota

**What:** Pay-per-call; the free analyzer spends real money per run, and a viral share or
a bot could burn budget.

**Mitigation:** D10 controls (3/IP/day, global 50/day kill-switchable via env, honeypot),
30-day details cache per place, field masks requesting only used fields, and a GCP budget
alert (owner task, note in LAUNCH-CHECKLIST). Cost ceiling at the launch cap is a few
US$/day worst case.

## R7 — PSI flakiness

**What:** PSI regularly takes 20–30 s or 500s under load; per-origin quotas apply.

**Mitigation:** Own pipeline step, 60 s budget, failure → skip with `PSI_UNAVAILABLE`
(never scored against the business, never fails the run) — PR-08 acceptance criteria
enforce this.

## R8 — Owner-data gaps for prospect analysis

**What:** Without profile access (prospects, by definition), Places exposes only ≤5
reviews and no posts data. Overclaiming precision would make reports factually wrong —
and the report is the sales asset.

**Mitigation:** Decisions D4/D5: posts recency excluded from automated scoring (manual
admin field), response rate framed as "visible reviews" observation. Copy in
`findings.ts` written to be true under partial data; native QA reviewer briefed on this.

## R9 — WhatsApp Cloud API (deliberately deferred)

**What:** Operator WhatsApp notifications would need Meta business verification, a
dedicated number, and template approval — weeks of process for one recipient.

**Mitigation:** D8: v1 notifies via SMTP email behind a `Notifier` interface; wa.me links
(no API) cover all client-facing WhatsApp needs. Revisit only if the operator misses
reviews in practice.

## R10 — Native-Spanish QA is a launch gate

**What:** All client-facing copy must read as natural Paraguayan Spanish; the builders
(LLMs) are not the QA. Skipping this ships a credibility risk in every report.

**Mitigation:** Copy is 100 % centralized in `src/copy/` (D2) so review is one sitting;
the QA step + sign-off is an explicit item in `docs/LAUNCH-CHECKLIST.md` (PR-17) and the
process is defined in COPY-GUIDE.md. Owner task: book the reviewer during Phase A, not at
the end.
