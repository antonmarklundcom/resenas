# Resenas — v1 Build Plan (Master)

**Product:** Resenas (resenas.com.py) — local-marketing software for Paraguayan SMBs.
**Free front end:** automated online-presence analyzer → Spanish report (web + PDF) → WhatsApp lead.
**Paid software function:** GBP management (auto-posting + review management) for clients marked as paying.
**Stack (fixed):** Next.js 15 (App Router) · Drizzle ORM · MySQL · Hostinger managed Node.js hosting.

> Note for executing sessions: if the skills `nodejs-mysql-hostinger-stack` and
> `nextjs-deploy-hostinger` are available in your environment, load them and follow their
> conventions. They were not present in the planning session; the Hostinger constraints they
> encode (no worker process, cron via HTTP-triggered API routes, managed Node limits) are
> baked into this plan.

---

## 1. Architecture overview

A single Next.js 15 monolith. No separate services, no worker process, no message queue —
Hostinger managed Node hosting runs exactly one Node process serving HTTP.

```
┌────────────────────────────── Next.js 15 (App Router) ──────────────────────────────┐
│                                                                                      │
│  Public routes                     API routes                    Admin routes        │
│  /            marketing home       /api/analyses  (create)       /admin (Auth.js     │
│  /analizar    analyzer input       /api/analyses/[id] (poll +    credentials,        │
│  /informe/[slug] web report          step-advance)               operator-only)      │
│  /informe/[slug]/pdf PDF stream    /api/leads                                        │
│                                    /api/cron/* (secret-gated)                        │
│                                                                                      │
│  Core modules (src/lib)                                                              │
│  analyzer/   places, gbp-checks, website-crawler, psi, instagram, scoring            │
│  report/     findings→copy mapping, web report data, PDF renderer (@react-pdf)       │
│  copy/       ALL client-facing Spanish text as typed, parameterized blocks           │
│  gbp/        OAuth, Business Profile API client, posting, reviews (Phase B)          │
│  ai/         Claude API wrappers for post drafts + review reply drafts (Phase B)     │
│  jobs/       step executor, job locks, cron handlers                                 │
└──────────────────────────────────────┬───────────────────────────────────────────────┘
                                       │ Drizzle ORM
                                  MySQL (Hostinger)
```

**External services:** Google Places API (New), PageSpeed Insights API, Google Business
Profile APIs (Phase B), Claude API (Phase B drafting), SMTP (operator notifications).

### Long-running work without a worker (the load-bearing pattern)

An analysis run takes 30–90 s end-to-end (crawl + PSI). Hostinger's managed Node proxy
cannot be trusted to hold a request that long, so **no single request ever runs the whole
pipeline**. Instead:

1. `POST /api/analyses` creates an `analysis_runs` row with `status = pending` and a
   `next_step` pointer. Returns immediately.
2. The client shows a progress screen and polls `GET /api/analyses/[id]` every ~2.5 s.
   **The poll endpoint is also the executor:** if the run has a pending step and no other
   request holds the row lock (`locked_until` column, compare-and-set via
   `UPDATE … WHERE locked_until < NOW()`), it executes exactly one step — each step is
   bounded to ~20 s — then returns current status. Steps: `places_fetch` →
   `website_crawl` → `psi` → `instagram` → `score` → `complete`.
3. A cron sweeper (`/api/cron/sweep-runs`, every 5 min) advances runs abandoned mid-flight
   (user closed the tab) and fails runs stuck > 15 min, so every run terminates.

This same lock-and-step pattern is reused in Phase B for post publishing and review polling.
It is the only concurrency mechanism in the system — do not introduce another.

### Cron on Hostinger

Hostinger cron (hPanel) can only run commands/curl on a schedule. All scheduled work is an
HTTP `GET`/`POST` to `/api/cron/<job>` with header `Authorization: Bearer ${CRON_SECRET}`.
Every cron route: (a) validates the secret, (b) acquires a named lock in `job_locks`,
(c) does a bounded amount of work (respecting request timeout), (d) records the outcome.
Jobs are idempotent and resumable — a run that processes 10 of 30 items finishes the rest
on the next tick.

Cron schedule (Phase B adds the last three):

| Route | Schedule | Purpose |
|---|---|---|
| `/api/cron/sweep-runs` | */5 min | advance/fail abandoned analysis runs |
| `/api/cron/publish-posts` | */10 min | publish approved GBP posts due now |
| `/api/cron/poll-reviews` | hourly | fetch new reviews for connected clients |
| `/api/cron/refresh-tokens` | daily | proactively refresh OAuth tokens near expiry |

---

## 2. Data model (Drizzle schema sketch)

Names are final; column details may grow during implementation but not shrink.

```ts
// ---------- Phase A ----------

businesses = mysqlTable('businesses', {
  id: bigint pk autoincrement,
  placeId: varchar(255) unique,          // Google place_id when resolved
  name: varchar(255) notNull,
  city: varchar(120),
  address: varchar(500),
  phone: varchar(40),                    // as published on GBP
  websiteUrl: varchar(500),
  instagramHandle: varchar(120),
  lat: decimal(10,7), lng: decimal(10,7),
  categoriesJson: json,                  // Places types + primary category
  createdAt, updatedAt,
});

analysisRuns = mysqlTable('analysis_runs', {
  id: bigint pk,
  businessId: fk -> businesses,
  reportSlug: varchar(32) unique notNull,   // unguessable, for shareable URL
  status: mysqlEnum(['pending','running','complete','failed']),
  nextStep: mysqlEnum(['places_fetch','website_crawl','psi','instagram','score','done']),
  lockedUntil: datetime,                    // step-executor row lock
  inputRaw: json,                           // what the user typed (URL / name+city / IG)
  gbpData: json, websiteData: json, instagramData: json, psiData: json,
  scoreTotal: tinyint,                      // 0–100
  scoreGbp: tinyint, scoreWebsite: tinyint, scoreInstagram: tinyint,
  findingsJson: json,                       // ordered array of {code, params, severity}
  instagramDegraded: boolean default false, // public fetch failed → manual/skipped
  error: text,
  createdAt, completedAt,
});

leads = mysqlTable('leads', {
  id: bigint pk,
  analysisRunId: fk -> analysis_runs,
  businessId: fk -> businesses,
  name: varchar(120) notNull,
  businessName: varchar(255) notNull,
  whatsapp: varchar(20) notNull,            // normalized E.164 (+595…)
  status: mysqlEnum(['nuevo','contactado','en_conversacion','propuesta','cliente','perdido'])
          default 'nuevo',
  isClient: boolean default false,          // mark-as-client (payment handled off-platform)
  source: varchar(60) default 'analyzer',
  createdAt, updatedAt,
});

leadNotes = mysqlTable('lead_notes', { id, leadId fk, body text, createdAt });

operators = mysqlTable('operators', { id, email unique, passwordHash, createdAt });

rateLimits = mysqlTable('rate_limits', {        // analyzer abuse control
  key: varchar(120) pk,                          // e.g. 'ip:1.2.3.4:2026-07-16'
  count: int, windowEndsAt: datetime,
});

jobLocks = mysqlTable('job_locks', {
  name: varchar(60) pk, lockedUntil: datetime, lastRunAt: datetime, lastStatus: varchar(20),
});

// ---------- Phase B ----------

gbpConnections = mysqlTable('gbp_connections', {
  id: bigint pk,
  businessId: fk -> businesses,
  leadId: fk -> leads,
  googleEmail: varchar(255),
  accountName: varchar(120),                // 'accounts/…'
  locationName: varchar(120) unique,        // 'locations/…'
  accessTokenEnc: text, refreshTokenEnc: text,   // AES-256-GCM, key in env
  tokenExpiresAt: datetime,
  status: mysqlEnum(['pending','active','revoked','error']),
  connectedAt, lastRefreshAt,
});

gbpPosts = mysqlTable('gbp_posts', {
  id: bigint pk,
  connectionId: fk -> gbp_connections,
  topic: varchar(255),                      // operator-supplied seed
  draftBody: text,                          // AI draft (Spanish)
  finalBody: text,                          // operator-edited/approved text
  ctaType: varchar(30), ctaUrl: varchar(500), mediaUrl: varchar(500),
  status: mysqlEnum(['draft','pending_approval','approved','published','failed','rejected']),
  scheduledFor: datetime,
  publishedAt: datetime, gbpPostName: varchar(255), error: text,
  createdAt, updatedAt,
});

gbpReviews = mysqlTable('gbp_reviews', {
  id: bigint pk,
  connectionId: fk -> gbp_connections,
  gbpReviewName: varchar(255) unique,       // idempotent upsert key for polling
  reviewerName: varchar(255), rating: tinyint, comment: text,
  reviewCreatedAt: datetime,
  replyDraft: text, replyFinal: text,
  replyStatus: mysqlEnum(['none','drafted','pending_approval','approved','published','failed']),
  repliedAt: datetime, notifiedAt: datetime, error: text,
  createdAt, updatedAt,
});
```

**Findings are codes, not prose.** The analyzer emits `{code, params}` (e.g.
`{code: 'GBP_FEW_PHOTOS', params: {count: 3}}`). The report layer maps codes to
pre-written, parameterized Spanish blocks in `src/copy/findings.ts`. This is what makes
the one-time native-speaker QA possible (see `COPY-GUIDE.md`) and keeps AI out of the
prospect-facing report entirely.

---

## 3. External APIs — approval, quotas, cost, risk

| API | Used for | Approval | Quota/cost reality | Risk level |
|---|---|---|---|---|
| **Places API (New)** | Business lookup + GBP-visible data (rating, review count, up to 5 reviews, photos, hours, attributes, categories) | None — enable + billing on a Google Cloud project | Pay-per-SKU. Text Search + Place Details (Pro/Enterprise fields) ≈ US$0.06–0.08 per full analysis. Cache aggressively (details cached 30 days on `businesses`); cap analyzer at 50 runs/day globally at launch | Low (cost, not access) |
| **PageSpeed Insights API** | Mobile performance signal | None — API key | Free; 25 000/day, 400/100 s. One call per analysis. PSI itself takes 15–30 s → its own pipeline step with 60 s budget and graceful skip on timeout | Low |
| **Business Profile APIs** (Phase B only) | Posts, review list + reply, location mgmt | **Formal application** (GCP project + business justification form). Historically days–weeks; default quota is 0 QPM until approved | After approval: generous for our volume (handful of clients) | **High — blocking for Phase B. Apply in week 1.** See RISKS.md |
| **Google OAuth** (Phase B) | Client grants profile access | OAuth consent screen verification (sensitive scope `business.manage`) — separate review from the API application | n/a | Medium — see RISKS.md |
| **Claude API** (Phase B) | Draft GBP posts + review replies (Spanish), operator-approved before anything is published | None | Haiku-class model; cents/month at v1 volume | Low |
| **SMTP (Hostinger mailbox)** | Operator notification on new reviews | None | Included in hosting | Low |
| **Instagram public data** | Lightweight presence check | None (no API used) | Unauthenticated fetch of the public profile page; unreliable by design → graceful degradation built in (see decision D7) | Medium — never blocking |

**PDF generation:** `@react-pdf/renderer` (pure JS — no headless Chromium, which cannot be
assumed on Hostinger managed Node). See decision D3.

---

## 4. Scoring model

Composite 0–100 = weighted per-area scores:

- **GBP: 50%** — completeness (description, categories, hours, phone, website link,
  attributes), photo count + recency, review count / rating / velocity (from Places-visible
  data), owner-response presence on visible reviews, NAP consistency vs website.
- **Website: 35%** — title/meta with local intent, H1, local keyword presence
  (city/neighborhood terms), `schema.org/LocalBusiness`, NAP present, WhatsApp CTA present,
  mobile viewport, SSL, indexability basics (robots, noindex, 200s), PSI mobile signal.
- **Instagram: 15%** — profile exists, bio has link/WhatsApp, post recency, follower count
  if retrievable.

**Reweighting when inputs are missing:** no website → GBP 77 / IG 23; no IG (or degraded)
→ GBP 59 / Web 41; only GBP → GBP 100%. The report always states what was and wasn't
evaluated (as opportunity framing: "aún no evaluamos tu sitio web…"). Weights live in one
config file; every rule contributes `{code, points, maxPoints}` so per-area scores are
auditable and findings fall out of the same pass.

---

## 5. Phasing

**Phase A — lead machine (PR-01 … PR-17).** Analyzer, scoring, report (web + PDF), lead
capture, admin panel, marketing site, production hardening. **Shippable and generating
WhatsApp leads with zero Google approvals beyond enabling Places billing.**

**Phase B — GBP management (PR-18 … PR-24).** OAuth + Business Profile API, client
onboarding, cron infra, auto-posting with approval queue, review management with approval
queue + operator notification. Gated on Google API approval, which is why the **Business
Profile API application and OAuth consent verification are submitted during week 1 of
Phase A** (external dependency, not a PR).

---

## 6. Decisions (resolved — do not reopen during implementation)

**D1 — Lead gate: teaser-then-gate.** The analyzer runs without any form. The results
screen shows the composite score, the per-area gauge, and exactly one concrete negative
finding — then gates the full report (all findings, "qué arreglaríamos", PDF) behind
name + business + WhatsApp. *Why:* asking for a phone number before showing anything kills
conversion on mobile; showing everything kills the reason to leave a number. A real score
plus one visible problem is the strongest possible form-fill motivator, and the WhatsApp
number is the only asset the business actually needs from this flow.

**D2 — Report copy is 100 % templated.** Findings are codes mapped to parameterized
Spanish blocks. No LLM text in any prospect-facing report. *Why:* hard requirement for
one-time native QA; also removes latency, cost, and tone risk from the free funnel.

**D3 — PDF via `@react-pdf/renderer`.** Pure-JS PDF, shared design tokens with the web
report, streamed from `/informe/[slug]/pdf` and cached best-effort on disk. *Why:* headless
Chromium is not a safe assumption on Hostinger managed Node (memory + binary availability),
and an external PDF API adds a paid dependency to the free funnel. The PDF is a designed
document, not a webpage screenshot — react-pdf is the right tool anyway.

**D4 — GBP "posts recency" is not scored for prospects.** Google removed public post data
from Places API; there is no legitimate automated way to read a stranger's posts. The
analyzer omits it from scoring; the admin panel has an optional manual field ("última
publicación vista") the operator can fill after eyeballing the profile, which then appears
in the report as an extra finding. *Why:* scraping Google SERPs is fragile and ToS-hostile;
degrading honestly is cheaper than a scraper we'd babysit forever.

**D5 — Owner response rate is approximated** from the ≤5 reviews Places returns (how many
have owner replies). The report words it as observed behavior ("de tus reseñas más
visibles, X sin respuesta"), never as an exact rate. Full review data arrives in Phase B
for actual clients via the Business Profile API.

**D6 — Async = poll-driven step executor + cron sweeper** (§1). No queue library, no
in-process `setInterval`, no reliance on `after()` surviving the proxy. *Why:* it's the
only pattern that is fully restart-safe on a single managed Node process.

**D7 — Instagram check is best-effort unauthenticated fetch with built-in degradation.**
One fetch of the public profile page with a strict 8 s timeout; parse what's parseable
(existence, bio, follower count from meta tags). Any failure sets `instagramDegraded`,
reweights the score, and the report shows a neutral "no pudimos verificar tu Instagram
automáticamente" line. The admin panel exposes manual override fields. *Why:* Meta offers
no API for arbitrary public profiles without the account's authorization; planned
degradation beats pretending otherwise.

**D8 — Operator notifications via email (SMTP), not WhatsApp Cloud API, in v1.** New-review
alerts go to the operator's inbox through Hostinger SMTP, behind a `Notifier` interface.
*Why:* WhatsApp Cloud API requires Meta business verification, template approval, and a
dedicated number — days-to-weeks of setup to notify exactly one person (the operator) who
already lives in email + admin panel. The interface makes a later swap a one-file change.
(Client-facing WhatsApp CTAs are unaffected — those are `wa.me` links, no API.)

**D9 — Admin auth: Auth.js (next-auth v5) credentials provider**, single `operators` row,
bcrypt hash, middleware-protected `/admin`. No sign-up, no roles, no OAuth login. *Why:*
one operator; smallest correct thing with CSRF/session handling we don't hand-roll.

**D10 — Abuse/cost control on the free analyzer:** 3 runs/IP/day + global 50 runs/day
(both in `rateLimits`, both env-tunable), Places details cached 30 days per `placeId`,
honeypot field + minimal same-origin check on the analyze endpoint. *Why:* every run costs
real Places/PSI money; a cap that a legitimate prospect never hits is free insurance.

**D11 — AI drafting (Phase B) uses the Claude API with fixed Spanish prompt templates**;
drafts land in `pending_approval` and nothing reaches Google without an explicit operator
approval click. Model: cheapest current Haiku-class; prompts live in `src/ai/prompts.ts`
and follow COPY-GUIDE rules. *Why:* human-in-the-loop is the suspension-risk and
brand-voice control; the drafts only ever save the operator typing time.

**D12 — Token security:** GBP OAuth tokens encrypted at rest (AES-256-GCM, key in env,
never logged), refresh handled centrally in the API client with single-flight per
connection, `status='revoked'` on invalid_grant with an operator email. *Why:* these
tokens control real businesses' public profiles; leakage or silent expiry are the two
failure modes that matter.

**D13 — Repo layout:** single app, `src/app` (routes), `src/lib` (modules per §1),
`src/copy` (all client-facing text), `src/db` (Drizzle schema + migrations), `drizzle/`
(generated SQL). Tests colocated `*.test.ts`, run with Vitest. Deploy = `next build` +
`next start` per Hostinger managed Node conventions; `drizzle-kit migrate` run as a
release step.
