# Resenas — v1 PR Breakdown (ordered)

Rules for executors:
- One PR = one branch = one model session, completed end-to-end (code + tests + green build).
- Base every branch on `main` at the tip of the previous merged PR. Dependencies listed are hard.
- All client-facing text goes through `src/copy/` — no literal Spanish strings in components.
- Phase A (PR-01…PR-17) must be fully shipped before any PR-18+ work starts; Phase B is
  gated on Google Business Profile API approval (submitted as an external task in week 1,
  see RISKS.md R1/R2).
- Model assignments follow: **sonnet-5** = scaffolding, CRUD, UI from clear specs, config,
  marketing pages; **opus-4.8** = scoring engine, external API integrations, PDF/report
  pipeline, cron/queue logic, security surface, tricky edge cases.

---

## Phase A — Analyzer, report, leads, marketing (shippable lead machine)

### PR-01 — Project scaffold + Hostinger deploy baseline
- **Branch:** `feat/pr-01-scaffold`
- **In:** Next.js 15 (App Router, TS strict), Tailwind, Drizzle + mysql2 wiring, env
  validation module (zod) listing every env var used in v1, `/api/health` (checks DB),
  Vitest setup, base layout with brand tokens (colors/font), `docs/DEPLOY.md` with
  Hostinger managed-Node steps (build cmd, start cmd, env vars, migration release step).
  If the `nodejs-mysql-hostinger-stack` / `nextjs-deploy-hostinger` skills are available
  in the session, follow them.
- **Not:** any schema beyond a `_healthcheck` probe, any page content.
- **Accept:** `npm run build` clean; `/api/health` returns 200 with DB round-trip;
  `npm test` runs; env validation fails fast with a readable message on a missing var.
- **Deps:** none. **Model:** `sonnet-5` — scaffolding/config. **Size:** M

### PR-02 — Core Phase-A schema + migrations
- **Branch:** `feat/pr-02-core-schema`
- **In:** Drizzle tables `businesses`, `analysis_runs`, `leads`, `lead_notes`,
  `operators`, `rate_limits`, `job_locks` exactly per PLAN.md §2; generated migrations;
  seed script creating the operator user from env; typed query helpers.
- **Not:** Phase-B tables (`gbp_*`).
- **Accept:** `drizzle-kit migrate` applies cleanly to an empty MySQL; seed creates
  operator; insert/select round-trip test per table passes.
- **Deps:** PR-01. **Model:** `sonnet-5` — schema from a written spec. **Size:** S

### PR-03 — Copy system + findings catalog skeleton
- **Branch:** `feat/pr-03-copy-system`
- **In:** `src/copy/` module: typed `t(block, params)` accessor; parameterized block files
  (`common.ts`, `analyzer.ts`, `report.ts`, `findings.ts` with the full finding-code enum
  and placeholder-but-well-written Spanish per COPY-GUIDE); lint script
  (`npm run lint:copy`) that fails CI on any banned term from COPY-GUIDE.md appearing in
  `src/copy/` or `src/app/`; hardcoded-Spanish-outside-copy check (heuristic: accented
  strings in JSX).
- **Not:** final native-QA'd wording (that's the launch QA step), report layout.
- **Accept:** lint:copy catches a seeded banned term in a test fixture; every finding code
  referenced by later PRs exists in the enum; params type-checked.
- **Deps:** PR-01. **Model:** `sonnet-5` — structured content + tooling from spec. **Size:** M

### PR-04 — Places API client + business resolution
- **Branch:** `feat/pr-04-places-lookup`
- **In:** Places API (New) client with typed field masks; Google Maps URL parsing (all
  common URL shapes: `maps.google.*`, `goo.gl`/`maps.app.goo.gl` short links via redirect
  follow, `place_id` extraction, coordinates fallback); Text Search for name+city scoped
  to Paraguay; Place Details fetch (rating, review count, ≤5 reviews w/ owner replies,
  photos metadata, hours, attributes, categories, phone, website); upsert into
  `businesses` with 30-day details cache; cost-guard: never fetch details twice inside
  cache TTL.
- **Not:** any scoring, any UI.
- **Accept:** unit tests for every URL shape (fixtures); integration test (mocked HTTP)
  for search→details→upsert; cache hit skips the details call (asserted); ambiguous
  search returns candidate list (data shape for PR-11's picker).
- **Deps:** PR-02. **Model:** `opus-4.8` — external API, URL-parsing edge cases, cost
  guards. **Size:** M

### PR-05 — Analysis run pipeline (state machine + step executor)
- **Branch:** `feat/pr-05-run-pipeline`
- **In:** `POST /api/analyses` (creates run, honeypot + same-origin check, rate limiting
  per PLAN D10); `GET /api/analyses/[id]` poll-and-advance endpoint with `locked_until`
  compare-and-set row lock; step registry (steps are pluggable — this PR ships
  `places_fetch` wired to PR-04 and no-op stubs for the rest); per-step timeout + error
  capture; `/api/cron/sweep-runs` with `job_locks` + `CRON_SECRET` auth; run terminal
  states guaranteed (stuck >15 min → failed).
- **Not:** the analyzers themselves (PR-06…PR-10 fill the stubs), any UI.
- **Accept:** concurrency test — two parallel poll requests execute a step exactly once;
  abandoned run is completed by sweeper in test; rate limit returns 429 with copy-block
  message; failed step records error and run reaches `failed`, never wedges.
- **Deps:** PR-02, PR-04. **Model:** `opus-4.8` — concurrency/cron logic, the riskiest
  code in Phase A. **Size:** L

### PR-06 — GBP analyzer (Places-visible data)
- **Branch:** `feat/pr-06-gbp-analyzer`
- **In:** `website_crawl`-independent GBP checks as a pure function over Place Details:
  completeness (description, categories, hours, phone, website link, attributes), photo
  count/recency, review count/rating/velocity (velocity from the ≤5 visible review
  timestamps, clearly approximate), owner-response presence (PLAN D5); emits
  `{code, params, points}` findings; stores `gbpData` on the run; wired as pipeline step.
- **Not:** posts recency (PLAN D4 — manual admin field comes in PR-15), NAP comparison
  (needs website data, lives in PR-07).
- **Accept:** table-driven unit tests: complete profile scores high, empty profile emits
  the full finding set; each check individually toggleable in fixtures; no network in tests.
- **Deps:** PR-05. **Model:** `opus-4.8` — scoring-adjacent rule engine. **Size:** M

### PR-07 — Website local-SEO analyzer (crawler)
- **Branch:** `feat/pr-07-website-analyzer`
- **In:** Bounded crawler (home + up to 3 same-origin contact/about-ish pages, 10 s/page
  timeout, 2 MB cap, real UA, redirects followed, HTTPS check); checks: title/meta with
  local intent, H1 presence/quality, local keyword presence (city terms), `LocalBusiness`
  JSON-LD/microdata, NAP extraction + consistency vs `businesses` (phone normalization to
  +595 formats, fuzzy address match), WhatsApp CTA detection (`wa.me`, `api.whatsapp.com`,
  tel-like patterns), mobile viewport meta, robots/noindex/basic indexability; emits
  findings; handles no-website runs (step skips, reweight flag).
- **Not:** PSI (PR-08), JS-rendered SPA execution (fetch + parse only; a JS-only site
  yields honest "no pudimos leer el contenido" finding).
- **Accept:** fixture-HTML tests per check (present/absent/malformed); phone normalization
  matrix (`021`, `+595 21`, `0981…`); crawler respects page/byte/time caps (asserted);
  dead site → findings, not crash.
- **Deps:** PR-05. **Model:** `opus-4.8` — crawling + parsing edge cases galore. **Size:** L

### PR-08 — PageSpeed Insights step
- **Branch:** `feat/pr-08-psi`
- **In:** PSI API client (mobile strategy), 60 s step budget, extract performance score +
  LCP; timeout/quota-error → graceful skip with `PSI_UNAVAILABLE` flag (not a finding
  against the business); wired as pipeline step; contributes to website score per §4.
- **Not:** desktop strategy, full Lighthouse detail storage (keep score + 3 key metrics).
- **Accept:** mocked-API tests: success, timeout→skip, quota-error→skip; run completes
  either way; score contribution matches spec table.
- **Deps:** PR-07. **Model:** `opus-4.8` — external API with slow/flaky failure modes. **Size:** S

### PR-09 — Instagram lightweight check
- **Branch:** `feat/pr-09-instagram`
- **In:** Single unauthenticated fetch of `instagram.com/<handle>` (8 s timeout); parse
  existence (404/redirect-to-login handling), og/meta tags for follower count + bio when
  present; per PLAN D7 any failure → `instagramDegraded=true`, reweighting, neutral copy;
  handle normalization (`@x`, URL, bare); wired as pipeline step.
- **Not:** Meta APIs, login, post-content analysis, retries beyond one.
- **Accept:** fixture tests: public profile parsed; login-wall response → degraded (not
  "no existe"); missing handle → step skipped; reweighting asserted in scoring fixtures.
- **Deps:** PR-05. **Model:** `opus-4.8` — unreliable-source handling is the whole point. **Size:** M

### PR-10 — Scoring engine + findings selection
- **Branch:** `feat/pr-10-scoring`
- **In:** `score` step: aggregates PR-06/07/08/09 outputs into per-area 0–100 + composite
  with the §4 weights and reweighting rules (single config file); orders findings by
  severity×impact; selects the one "teaser finding" (PLAN D1: highest-impact negative);
  persists all score fields + `findingsJson`; golden-file tests.
- **Not:** any rendering.
- **Accept:** golden fixtures (strong/mediocre/weak/GBP-only/no-IG businesses) produce
  exact expected scores; reweighting sums to 100; teaser selection deterministic; changing
  a weight breaks only the config + goldens.
- **Deps:** PR-06, PR-07, PR-08, PR-09. **Model:** `opus-4.8` — the scoring engine. **Size:** M

### PR-11 — Analyzer UI (input + progress)
- **Branch:** `feat/pr-11-analyzer-ui`
- **In:** `/analizar` page: mobile-first input (Maps URL **or** name+city tabs, optional
  website + Instagram fields), candidate picker when search is ambiguous (PR-04 shape),
  progress screen polling `GET /api/analyses/[id]` with per-step Spanish status lines,
  friendly failure state with WhatsApp fallback CTA; all text from `src/copy`.
- **Not:** results/teaser display (PR-12), marketing homepage (PR-16).
- **Accept:** happy path e2e (mocked pipeline) from input → progress → redirect to
  results; ambiguous search shows picker; failure state renders; Lighthouse mobile
  usability ≥ 90 on the input page; works on 360 px viewport.
- **Deps:** PR-05 (API shapes), PR-03. **Model:** `sonnet-5` — UI from a clear spec. **Size:** M

### PR-12 — Teaser + lead gate
- **Branch:** `feat/pr-12-lead-gate`
- **In:** Results teaser view (composite score gauge, three area chips, the one teaser
  finding) + gate form (name, business, WhatsApp) per PLAN D1; WhatsApp normalization to
  E.164 with `+595` default + validation; `POST /api/leads` (ties lead → run → business,
  idempotent per run); on submit → unlock full report route; server-side gating (report
  page checks a signed cookie/lead existence — the slug alone shows teaser+gate).
- **Not:** the full report content (PR-13), admin views.
- **Accept:** gate cannot be bypassed by hitting the report URL cold (server-checked);
  phone matrix tests (`0981…`, `+595981…`, garbage → inline error); duplicate submit for
  same run doesn't duplicate leads; lead row lands with `status='nuevo'`.
- **Deps:** PR-10, PR-11. **Model:** `sonnet-5` — form/CRUD from spec (gating check is
  specified exactly). **Size:** M

### PR-13 — Web report page
- **Branch:** `feat/pr-13-web-report`
- **In:** `/informe/[slug]`: branded, mobile-first, forward-worthy report — hero with
  composite score + business name, per-area sections with outcome-framed findings (all
  copy from `src/copy/findings.ts`), "Qué arreglaríamos primero" top-3 section, what-we
  -couldn't-evaluate notes (degraded IG / no website), sticky WhatsApp CTA
  (`wa.me/<operator>?text=` prefilled with business name), PDF download button
  (links PR-14 route), OG tags so the shared link previews well on WhatsApp.
- **Not:** PDF itself, any English, any SEO-jargon leak (lint:copy enforces).
- **Accept:** renders all golden fixtures from PR-10 without layout breaks at 360 px;
  every finding code has copy (build fails on a missing block); WhatsApp CTA carries
  business name; OG preview validated; gate from PR-12 respected.
- **Deps:** PR-12. **Model:** `sonnet-5` — demanding but fully specified UI. **Size:** L

### PR-14 — PDF report pipeline
- **Branch:** `feat/pr-14-pdf`
- **In:** `@react-pdf/renderer` document mirroring the web report's structure/brand
  (shared token file, embedded font with full Spanish glyphs); `GET /informe/[slug]/pdf`
  streams the PDF (gated same as web report); best-effort disk cache keyed by run id +
  copy version; memory-bounded rendering.
- **Not:** headless-browser rendering, per-report AI text, email delivery.
- **Accept:** golden fixtures render to valid PDFs (parsed page count + text-extraction
  spot checks incl. `ñ`/accents); cold generation < 5 s locally; cache hit skips render;
  gate enforced.
- **Deps:** PR-13. **Model:** `opus-4.8` — PDF pipeline (layout engine quirks, fonts,
  memory). **Size:** M

### PR-15 — Admin panel
- **Branch:** `feat/pr-15-admin`
- **In:** Auth.js credentials login per PLAN D9 (middleware-protected `/admin`); leads
  list (filter by status, search, WhatsApp deep-link per lead), lead detail (status
  select incl. `cliente`, `isClient` toggle, notes CRUD, linked run/report links);
  analysis-runs list (status, score, re-run action, link to report); manual GBP
  posts-recency field per PLAN D4 + IG manual-override fields per D7 (re-scores on save);
  plain server-rendered tables — operator-only, function over polish.
- **Not:** client-facing anything, roles, audit log, Phase-B screens.
- **Accept:** unauthenticated `/admin/*` redirects to login; lead status/notes/mark-as-
  client round-trip; manual overrides trigger re-score (asserted); WhatsApp deep-link
  correct; usable on a phone.
- **Deps:** PR-12 (leads exist); PR-10 (re-score). **Model:** `sonnet-5` — auth-from-
  recipe + CRUD. **Size:** L

### PR-16 — Marketing site
- **Branch:** `feat/pr-16-marketing-site`
- **In:** Homepage (Merchynt-style structure/energy, analyzer as hero: input embedded,
  submits into the `/analizar` flow), how-it-works (3 steps), sample-report section
  (screenshots/live demo slug), services overview (static: web, contenido, Google Ads,
  Meta Ads, automatización, CRM — as conversation starters, WhatsApp CTA each),
  social-proof placeholder block, FAQ, footer; floating WhatsApp button site-wide;
  metadata/OG/sitemap/robots; all copy in `src/copy`.
- **Not:** blog, pricing page, English, any backend changes.
- **Accept:** Lighthouse mobile perf ≥ 85 / SEO ≥ 95 on `/`; hero input reaches analyzer
  flow; every CTA resolves to `wa.me` or analyzer; lint:copy passes; 360 px clean.
- **Deps:** PR-11 (flow), PR-03. **Model:** `sonnet-5` — marketing pages. **Size:** L

### PR-17 — Production hardening + launch checklist
- **Branch:** `feat/pr-17-hardening-launch`
- **In:** Security headers (CSP compatible with the app, HSTS, frame/nosniff), request
  logging with secret redaction, structured error handler (no stack leaks), analyzer
  abuse controls verified end-to-end (D10) + global daily budget kill-switch env,
  `/api/cron/sweep-runs` documented in `docs/DEPLOY.md` cron table with exact hPanel
  entries, DB backup note, uptime-check endpoint, `docs/LAUNCH-CHECKLIST.md` (incl. the
  native-speaker copy QA sign-off step from COPY-GUIDE.md and the Google API applications
  status check from RISKS.md).
- **Not:** Phase-B code.
- **Accept:** headers verified in integration test; kill-switch returns friendly 503 copy
  block; checklist complete with owner per item; no secret appears in logs (test greps a
  captured log fixture).
- **Deps:** all Phase A. **Model:** `opus-4.8` — security surface. **Size:** M

**→ Phase A ships here. Lead gen live while Google approval (R1/R2) is pending.**

---

## Phase B — GBP management (gated on Google API approval)

### PR-18 — Google OAuth + Business Profile API client
- **Branch:** `feat/pr-18-google-oauth`
- **In:** `gbp_connections` (+ Phase-B tables `gbp_posts`, `gbp_reviews`) migrations;
  OAuth flow (offline access, `business.manage` scope) on dedicated `/conectar/*` routes;
  AES-256-GCM token encryption per PLAN D12; central BP API client (account/location
  listing, single-flight token refresh, invalid_grant → `revoked` + operator email,
  quota-aware retries with backoff); location picker when the Google account manages
  several; connection status surfaced.
- **Not:** posting/review features, onboarding UX polish (PR-19), cron.
- **Accept:** mocked-OAuth integration tests: happy path stores encrypted tokens
  (ciphertext asserted ≠ plaintext, decrypts round-trip); refresh single-flight under
  parallel calls; revoked token → status + notification; location listing/picker works
  with 0/1/N locations.
- **Deps:** Phase A complete; **external:** R1 + R2 approved. **Model:** `opus-4.8` —
  OAuth + token security. **Size:** L

### PR-19 — Client onboarding flow
- **Branch:** `feat/pr-19-client-onboarding`
- **In:** Operator generates a per-client invite link from admin (tokenized, expiring);
  `/conectar/[token]`: mobile-first guided flow for a non-technical client arriving from
  WhatsApp — plain-Spanish explanation of what access is granted and why, big single
  button into PR-18's OAuth, confusion-proof error states (wrong Google account: shows
  which account they used + retry; no GBP on account: explains + WhatsApp help CTA),
  success screen ("listo, nosotros nos encargamos") + operator email on completion;
  connection appears linked to the lead/business in admin.
- **Not:** the OAuth mechanics themselves (PR-18), any self-serve settings.
- **Accept:** e2e (mocked OAuth): invite → grant → connection linked to correct lead;
  expired/used token → friendly dead-end with WhatsApp CTA; wrong-account path renders;
  copy through `src/copy` + lint passes; flawless at 360 px.
- **Deps:** PR-18. **Model:** `sonnet-5` — guided UX from a precise spec. **Size:** M

### PR-20 — Cron infrastructure (Phase-B general runner)
- **Branch:** `feat/pr-20-cron-infra`
- **In:** Shared cron-route factory (secret auth, `job_locks` acquire/release, bounded
  batch size, per-item error isolation — one bad item never kills the batch, structured
  result logging, resumability); `/api/cron/refresh-tokens` as first consumer; DEPLOY.md
  cron table updated with all Phase-B schedules.
- **Not:** posting/review business logic (PR-21/23 plug in as consumers).
- **Accept:** lock prevents overlapping executions (parallel-invocation test); item
  failure isolates + records, batch continues; unauthorized call → 401; token-refresh
  consumer refreshes only near-expiry connections (fixture matrix).
- **Deps:** PR-18. **Model:** `opus-4.8` — cron/locking logic. **Size:** M

### PR-21 — Auto-posting engine (drafting + scheduling + publishing)
- **Branch:** `feat/pr-21-posting-engine`
- **In:** Claude API drafting service (fixed Spanish prompt templates per PLAN D11, input:
  business info + operator topic; output validated: length caps, banned-terms lint reused
  from PR-03); post lifecycle per `gbp_posts` states (drafts land `pending_approval`;
  **nothing publishes without `approved`**); `/api/cron/publish-posts` (due `approved` →
  BP API create post, idempotency guard against double-publish, failure → `failed` +
  error + operator email, retry policy: 3 attempts then park).
- **Not:** the approval UI (PR-22 — this PR ships service + minimal test harness), media
  upload beyond a URL field.
- **Accept:** draft generation mocked-LLM test (validation rejects overlong/banned
  output); state machine enforces approval gate (attempt to publish `draft` → refused);
  double-cron-tick publishes exactly once; failure path parks + notifies.
- **Deps:** PR-20. **Model:** `opus-4.8` — external APIs + publish-exactly-once. **Size:** L

### PR-22 — Posts admin UI (approval queue)
- **Branch:** `feat/pr-22-posts-admin-ui`
- **In:** Admin section per connected client: request-draft form (topic + CTA), pending-
  approval queue with inline edit (edited text → `finalBody`), approve+schedule
  (date/time), reject with reason; published/failed history with error display + retry
  button; per-client monthly calendar-ish list view.
- **Not:** engine changes, client-facing anything.
- **Accept:** full lifecycle drivable from UI in e2e (draft → edit → approve → visible as
  scheduled → mocked publish → history); reject removes from queue; retry re-queues a
  `failed` post; phone-usable.
- **Deps:** PR-21. **Model:** `sonnet-5` — CRUD/UI on a finished engine. **Size:** M

### PR-23 — Review management engine (poll + AI drafts + publish)
- **Branch:** `feat/pr-23-review-engine`
- **In:** `/api/cron/poll-reviews` (hourly, per-connection fetch, idempotent upsert on
  `gbpReviewName`, batch-bounded); new-review → Claude API reply draft (Spanish, rating-
  aware tone: ≤2★ drafts are apologetic + take-it-offline-to-WhatsApp, 5★ warm + brief;
  same validation as PR-21) → `pending_approval`; publish-on-approve via BP API reply
  endpoint (idempotent, failure → `failed` + operator email); `Notifier` interface (PLAN
  D8) with SMTP implementation — new-review email with rating, text, draft, admin
  deep-link; `notifiedAt` guard against duplicate emails.
- **Not:** approval UI (PR-24), auto-publish without approval (never), WhatsApp Cloud API.
- **Accept:** poll idempotency (same review twice → one row, one email); rating-tone
  matrix on mocked LLM; approval gate enforced; reply publish exactly-once; notifier
  swap-tested via a fake implementation.
- **Deps:** PR-20 (+ PR-21's shared AI validation). **Model:** `opus-4.8` — polling
  idempotency + API surface. **Size:** L

### PR-24 — Reviews admin UI + notification polish
- **Branch:** `feat/pr-24-reviews-admin-ui`
- **In:** Admin reviews inbox (per client + unified), each item: review, star rating,
  AI draft, inline edit, approve-and-publish button, skip/no-reply action; filter
  unanswered/answered; response-rate stat per client; email template polish (mobile
  inbox friendly); end-to-end walkthrough doc for the operator (`docs/OPERATOR-GUIDE.md`
  covering leads, posts, reviews workflows).
- **Not:** engine changes.
- **Accept:** e2e: review appears → edit draft → approve → mocked publish → marked
  answered; skip works; stats correct on fixtures; operator guide covers all three
  workflows; phone-usable.
- **Deps:** PR-23, PR-22. **Model:** `sonnet-5` — inbox UI on a finished engine. **Size:** M

---

## Summary table

| PR | Title | Model | Size | Hard deps |
|---|---|---|---|---|
| 01 | Scaffold + deploy baseline | sonnet-5 | M | — |
| 02 | Core schema | sonnet-5 | S | 01 |
| 03 | Copy system + findings catalog | sonnet-5 | M | 01 |
| 04 | Places client + resolution | opus-4.8 | M | 02 |
| 05 | Run pipeline (state machine) | opus-4.8 | L | 02, 04 |
| 06 | GBP analyzer | opus-4.8 | M | 05 |
| 07 | Website analyzer | opus-4.8 | L | 05 |
| 08 | PSI step | opus-4.8 | S | 07 |
| 09 | Instagram check | opus-4.8 | M | 05 |
| 10 | Scoring engine | opus-4.8 | M | 06–09 |
| 11 | Analyzer UI | sonnet-5 | M | 03, 05 |
| 12 | Teaser + lead gate | sonnet-5 | M | 10, 11 |
| 13 | Web report | sonnet-5 | L | 12 |
| 14 | PDF pipeline | opus-4.8 | M | 13 |
| 15 | Admin panel | sonnet-5 | L | 10, 12 |
| 16 | Marketing site | sonnet-5 | L | 03, 11 |
| 17 | Hardening + launch | opus-4.8 | M | Phase A |
| 18 | Google OAuth + BP client | opus-4.8 | L | 17 + approvals |
| 19 | Client onboarding flow | sonnet-5 | M | 18 |
| 20 | Cron infra | opus-4.8 | M | 18 |
| 21 | Posting engine | opus-4.8 | L | 20 |
| 22 | Posts admin UI | sonnet-5 | M | 21 |
| 23 | Review engine | opus-4.8 | L | 20, 21 |
| 24 | Reviews admin UI + guide | sonnet-5 | M | 22, 23 |

Parallelization notes: 03 ∥ 02; 06/07/09 ∥ each other after 05; 11 can start once 05's
API shapes merge; 15 ∥ 13–14; 16 ∥ 12–15. Phase B strictly after PR-17 ships.
