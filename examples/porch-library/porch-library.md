# Porch Library: Technical Plan

> Example output of PlanForge. The app idea is made up. Status: finalized after round 2 with `/finalize` (the council had not yet converged; see the end of this file).

## 1. App Architecture Framework

A server-rendered Next.js monolith with a small REST API. Map tiles come from OpenStreetMap for the pilot behind a TILE_URL switch, so a keyed provider can replace them without code changes.

### Decisions
- **Next.js 15 App Router, output: 'standalone'**: One deploy bundle that carries its own runtime dependencies.
- **Leaflet with a configurable TILE_URL (default OSM) and visible attribution**: OSM's tile policy allows light use with attribution but can block heavy use; the switch removes the single point of failure.

### Implementation

**frontend_framework**: Next.js 15

**frontend_language**: TypeScript (strict)

**backend_pattern**: Monolith with route handlers

**backend_framework**: Next.js route handlers

**api_style**: REST JSON: GET /api/libraries?bbox=, POST /api/libraries, PATCH|DELETE /api/libraries/:id, POST /api/libraries/:id/books, DELETE /api/books/:id, POST /api/libraries/:id/checkins, POST /api/reports, POST /api/uploads, GET /api/health, GET /api/geocode?q=

**repo_structure**: app/ and components/ (Lead Builder), db/ (Data Engineer), tests/ (QA), infra/ (DevOps), lib/ shared

**monorepo_tool**: none

**state_management**: Server Components plus TanStack Query for the map

**rendering_strategy**: SSR for library pages; client-only map via next/dynamic ssr:false

**key_libraries**
- leaflet
- react-leaflet
- drizzle-orm
- zod
- @tanstack/react-query
- sharp
- rate-limiter-flexible
- @sentry/nextjs

### Risks
- OSM blocks or throttles tile requests **Mitigation:** TILE_URL switch, attribution, a custom User-Agent-friendly Referer, and a documented swap to a keyed provider

### Agent notes
- All map code lives in components/map/ and reads TILE_URL from config.
- Every handler calls the shared auth helpers from lib/authz.ts.

## 2. Database Setup

PostgreSQL 16 with explicit status enums, Auth.js adapter tables, and expand/contract migrations. Backups go off the box.

### Decisions
- **Two database roles: migration owner and DML-only runtime role**: A compromised app cannot alter the schema.
- **Expand/contract migrations only**: Old code must keep working against the new schema during a deploy.

### Implementation

**primary_db**: PostgreSQL 16

**orm_or_client**: drizzle-orm + drizzle-kit

**schema_overview**
- table: users | purpose: Accounts | key_fields: id, email (lowercased, unique), display_name, role enum(member,admin), status enum(active,suspended), created_at
- table: accounts, sessions, verification_tokens | purpose: Auth.js adapter tables | key_fields: per @auth/drizzle-adapter
- table: libraries | purpose: A porch library | key_fields: id, owner_id, name, lat, lng, address_hint, photo_key, status enum(active,hidden,removed), created_at
- table: books | purpose: Titles in a library | key_fields: id, library_id, isbn, title, author, added_by, removed_at
- table: checkins | purpose: Visits and notes | key_fields: id, library_id, user_id, note, hidden, created_at
- table: reports | purpose: Reports on libraries, check-ins or users | key_fields: id, target_type, target_id, reporter_id, reason, resolved_by, resolved_at
- table: rate_limits | purpose: Postgres-backed limiter counters | key_fields: key, points, expire_at
- table: isbn_cache | purpose: Open Library results | key_fields: isbn, payload, fetched_at

**indexing_strategy**: Composite index on libraries(status, lat, lng) for bounding-box queries; books(library_id) where removed_at is null; checkins(library_id, created_at desc); plan earthdistance or PostGIS past 50k rows

**migration_strategy**: drizzle-kit generate; run in a dedicated deploy step by the migration role; expand/contract only

**backup_strategy**: Nightly pg_dump, encrypted with age, uploaded to a separate bucket with a write-only key, 30 day retention, alert if newest dump is older than 26 hours or under a minimum size, restore into a scratch database before launch and quarterly

**connection_pooling**: pg Pool max 10 for the runtime role

### Risks
- Backups silently stop **Mitigation:** Freshness and size alert, quarterly restore drill

### Agent notes
- Soft-delete books and hide libraries; never hard delete in handlers.
- Account deletion anonymizes check-ins and removes the email.

## 3. Server Requirements

One VPS runs production and a gated staging side by side, behind nginx, with a complete env var list.

### Decisions
- **Staging as a second pm2 app on the same VPS**: Cheapest topology; isolated by port, database, role, bucket prefix and basic auth.

### Implementation

**host**: Generic Linux VPS

**os**: Ubuntu 24.04 LTS

**cpu_ram_estimate**: 2 vCPU, 4 GB RAM, 60 GB disk (prod and staging)

**ports**
- port: 22 | service: SSH (key only, fail2ban)
- port: 80 | service: nginx (certbot and redirect)
- port: 443 | service: nginx
- port: 3000 | service: prod Next.js (127.0.0.1)
- port: 3001 | service: staging Next.js (127.0.0.1)
- port: 5432 | service: Postgres (127.0.0.1)

**nginx_config_notes**: TLS via certbot; set X-Real-IP from $remote_addr; client_max_body_size 64k globally and 6m only on /api/uploads; limit_req as a coarse outer layer; HSTS, X-Content-Type-Options, Referrer-Policy; staging vhost behind basic auth

**pm2_processes**
- porch-library (3000, cluster 2 instances)
- porch-library-staging (3001)
- pm2-logrotate module

**env_vars_needed**
- key: DATABASE_URL | purpose: Runtime DML-only role
- key: MIGRATE_DATABASE_URL | purpose: Migration role, used only in the deploy step
- key: AUTH_SECRET | purpose: Session and token signing
- key: AUTH_URL | purpose: Canonical site URL
- key: SMTP_URL | purpose: Postmark SMTP relay for Auth.js
- key: EMAIL_DAILY_CAP | purpose: Hard daily send ceiling
- key: S3_ENDPOINT, S3_REGION, S3_BUCKET, S3_ACCESS_KEY, S3_SECRET_KEY | purpose: Photo bucket, PutObject-only key
- key: PHOTO_PUBLIC_BASE | purpose: Public photo host used in CSP
- key: TILE_URL | purpose: Map tile source
- key: TURNSTILE_SITE_KEY, TURNSTILE_SECRET | purpose: Bot check on login and /add
- key: SENTRY_DSN, SENTRY_AUTH_TOKEN | purpose: Error reporting and source map upload (CI only)

**domain_routing**: porchlibrary.example (prod), staging.porchlibrary.example (basic auth)

### Risks
- Staging starves prod of memory **Mitigation:** pm2 max_memory_restart on staging and a 4 GB RAM alert

### Agent notes
- Bind everything except nginx to 127.0.0.1.
- Run the app as a non-root app user; deploy user has sudo only for pm2 reload.

## 4. Environment Strategy

Dev, staging and prod each have separate databases, buckets and secrets, with approval before prod.

### Decisions
- **Server-side env files plus GitHub environment secrets for CI**: Small team; no secret manager needed yet.

### Implementation

**environments**
- dev
- staging
- prod

**env_file_strategy**: .env.local for dev; /etc/porch-library/env and /etc/porch-library-staging/env, mode 600, owned by the app user; .env.example lists every key

**secrets_management**: Files on the server; CI deploy key scoped to the deploy user; rotation runbook per secret (AUTH_SECRET rotation logs everyone out, which is acceptable)

**feature_flag_approach**: Env booleans, for example FEATURE_PHOTOS

**promotion_process**: Merge to main deploys staging; prod tag requires GitHub environment approval

### Risks
- Staging sends real email **Mitigation:** Staging SMTP goes to a catch-all inbox and staging has its own Postmark stream

### Agent notes
- Never log or commit env files; gitleaks runs in CI.

## 5. Features

Find nearby libraries, view and change contents, and moderate. Criteria are written as Given/When/Then with numbers and a permissions matrix.

### Decisions
- **Photo optional, edit/delete and accessible list view in MVP**: Real users need to fix mistakes and use the app without the map.

### Implementation

**mvp_features**
- name: Browse map | description: Pins within the viewport; center defaults to the device location, else a typed address, else the pilot neighborhood | priority: high | acceptance_criteria: Given the 20-library seed and throttled 4G, when the page loads, then first pins render in under 3 s; given geolocation is denied, then an address search box is shown and the map centers on the default; results capped at 200 pins per request
- name: Nearby list view | description: Accessible text list of the same results, keyboard navigable | priority: high | acceptance_criteria: Given any map view, when the list tab is opened, then libraries are listed by distance and each is reachable by keyboard
- name: Library page | description: Photo, address hint, current books | priority: high | acceptance_criteria: Given a library with 60 books, when opened, then the first 50 show with a Load more control
- name: Add a library | description: Member picks a point on the map, names it, optionally adds a photo | priority: high | acceptance_criteria: Given a logged-in member, when they submit valid data, then it appears on the map within one refresh; given a library within 20 m, a duplicate warning is shown
- name: Edit or delete my library | description: Owner or admin edits fields or removes it | priority: high | acceptance_criteria: Given I am the owner, when I delete, then status becomes removed and the public page shows the gone page
- name: Add or remove a book | description: ISBN lookup or manual title and author | priority: high | acceptance_criteria: Given a member, when they add a book, then it shows after the server confirms; only the adder, the library owner or an admin can remove it
- name: Check-in note | description: Leave a note | priority: medium | acceptance_criteria: Given a member, a note up to 280 characters shows newest first
- name: Report | description: Report a library, check-in or user | priority: high | acceptance_criteria: Given a member, when they report, then it appears in /admin/reports
- name: Moderation | description: Admin hides a library, deletes a check-in, or suspends a user | priority: high | acceptance_criteria: Given an admin action, then the target is hidden for public users and the action is logged
- name: My page and account deletion | description: /me lists my libraries and check-ins and lets me delete my account | priority: medium | acceptance_criteria: Given I delete my account, then my email is removed and my check-ins are anonymized

**future_features**
- name: Wishlist alerts | description: Notify when a wanted title appears | phase: v2
- name: Pin clustering | description: Cluster dense areas | phase: v2
- name: Reading streaks | description: Gamified check-ins | phase: v2

### Risks
- Home addresses exposed **Mitigation:** Owner may set an approximate location and use a takedown path

### Agent notes
- Permissions matrix: visitor read; member add and edit own; owner edit and delete own library and any book in it; admin all.

## 6. Navigation Structure

Seven routes with a mobile tab bar. Admin routes return 404 to non-admins and login redirects back with ?next=.

### Decisions
- **Tabs: Map, List, Add, Me**: Covers the main jobs; admin is reached from Me for admins only.

### Implementation

**routes**
- path: / | component: MapPage | auth_required: false | description: Map and nearby list
- path: /library/[id] | component: LibraryPage | auth_required: false | description: Library with in-page forms for books, check-ins and reports
- path: /add | component: AddLibraryPage | auth_required: true | description: Create library with map pin picker
- path: /login | component: LoginPage | auth_required: false | description: Enter email, Turnstile
- path: /login/check-email | component: CheckEmailPage | auth_required: false | description: Check your inbox screen
- path: /me | component: ProfilePage | auth_required: true | description: My libraries, check-ins, delete account, logout
- path: /admin/reports | component: ReportsPage | auth_required: true | description: Admin queue; 404 for non-admins
- path: /privacy | component: PrivacyPage | auth_required: false | description: Privacy and terms

**layout_structure**: Root layout with TopBar, main, TabBar

**nav_components**
- TabBar (Map, List, Add, Me)
- TopBar

**deep_link_strategy**: Public library URLs with sanitized Open Graph tags

**404_handling**: Not-found page linking to the map; hidden or removed libraries show a gone page

### Risks
- Open redirect through ?next= **Mitigation:** Accept only same-origin relative paths

### Agent notes
- Add book, check-in and report are in-page forms posting to the API routes listed in architecture.

## 7. Authentication

Passwordless magic links through Auth.js with server-side authorization on every mutating handler.

### Decisions
- **Auth.js with Drizzle adapter and Postmark SMTP relay**: One mail path, standard tables.
- **Magic link lands on a POST confirm page**: Email scanners that prefetch links cannot consume the token.

### Implementation

**provider**: Auth.js v5 email provider with @auth/drizzle-adapter

**session_strategy**: Database sessions

**token_storage**: httpOnly secure SameSite=Lax cookie

**token_expiry**: Magic link 15 minutes, single use; session 30 days sliding

**refresh_strategy**: Sliding expiry on activity

**roles**
- name: visitor | permissions: Read public pages
- name: member | permissions: Create and edit own content, report
- name: admin | permissions: Resolve reports, hide libraries, delete check-ins, suspend users

**protected_routes**
- /add
- /me
- /admin/*

**oauth_providers**
- none in MVP

### Risks
- IDOR on mutating routes **Mitigation:** lib/authz.ts requireRole and requireOwnerOrAdmin on every handler, with a per-route test

### Agent notes
- Lowercase and trim emails before lookup and limiting.
- Suspended users are rejected at the session check.
- Admin role is set only by a direct database update.

## 8. 3rd-Party Tools & APIs

Postmark SMTP, an S3-compatible bucket, Open Library, OpenStreetMap tiles, a geocoder and Turnstile.

### Decisions
- **Photos are uploaded to our server, re-encoded with sharp, then written to the bucket**: Required to strip EXIF and validate content.

### Implementation

**integrations**
- name: Postmark (SMTP relay) | purpose: Magic-link email | sdk_or_package: nodemailer via Auth.js | auth_method: SMTP credentials in SMTP_URL | rate_limits: Plan limits; app enforces EMAIL_DAILY_CAP | fallback_strategy: Show retry later and alert
- name: S3-compatible bucket | purpose: Photos | sdk_or_package: @aws-sdk/client-s3 | auth_method: PutObject-only key to one prefix | rate_limits: None meaningful | fallback_strategy: Listing works without photos
- name: Open Library | purpose: ISBN lookup | sdk_or_package: fetch with 5 s timeout | auth_method: none | rate_limits: Polite use; 24 h cache in isbn_cache | fallback_strategy: Manual title and author
- name: OpenStreetMap tiles | purpose: Map | sdk_or_package: Leaflet | auth_method: none | rate_limits: Light use per OSM policy | fallback_strategy: Swap TILE_URL to a keyed provider
- name: Nominatim geocoder | purpose: Address search | sdk_or_package: fetch via GET /api/geocode | auth_method: none, identifying User-Agent | rate_limits: 1 request per second, cached | fallback_strategy: Pan the map manually
- name: Cloudflare Turnstile | purpose: Bot check | sdk_or_package: @marsidev/react-turnstile | auth_method: site and secret keys | rate_limits: none | fallback_strategy: Fail closed on login, with a retry

### Risks
- Postmark rejects the new account **Mitigation:** Request approval in week 1

### Agent notes
- Cap every outbound call at 5 s.
- Photo keys are server-generated UUIDs; clients never supply photo_url.

## 9. Security Considerations

Postgres-backed rate limits keyed on the real client IP, strict upload handling, nonce-based CSP, and authorization on every write.

### Decisions
- **Postgres-backed rate-limiter-flexible**: Survives restarts and works across cluster workers.

### Implementation

**rate_limiting**: tool: rate-limiter-flexible with Postgres store plus nginx limit_req | limits: Login 5/hour per email and 10/hour per IP; new accounts 3/day per IP; writes 30/hour per user; uploads 10/hour per user; reads 300/min per IP; global EMAIL_DAILY_CAP

**input_validation**: zod on every handler with max lengths (name 80, note 280, address_hint 120, title 200); photo uploads validated by magic bytes, 5 MB, sharp limitInputPixels 24 MP, 10 s timeout, EXIF stripped

**cors_policy**: Same-origin only

**csp_headers**: Nonce per request via middleware: default-src 'self'; script-src 'self' 'nonce-…' ; connect-src 'self' plus Sentry ingest host; img-src 'self' tile host and PHOTO_PUBLIC_BASE; frame-ancestors 'none'; base-uri 'self'; object-src 'none'

**sql_injection_prevention**: Drizzle parameterized queries

**xss_prevention**: React escaping, no dangerouslySetInnerHTML, escaped OG tags

**csrf_protection**: SameSite=Lax plus Origin check on writes

**dependency_audit_tool**: npm audit (high fails CI) plus Dependabot

**secrets_scanning**: gitleaks in CI

### Risks
- Spoofed client IP defeats IP limits **Mitigation:** App trusts X-Real-IP only from 127.0.0.1; integration test asserts distinct IPs count separately

### Agent notes
- Sentry sendDefaultPii false with a beforeSend scrubber for cookies, headers and bodies.

## 10. QA Testing Strategy

Vitest, API tests against real Postgres, and Playwright flows including authorization and limiter tests.

### Decisions
- **Authorization tests per mutating route**: IDOR is the most likely real bug.

### Implementation

**unit_testing**: tool: Vitest | coverage_target: 80% on lib/, enforced in CI

**integration_testing**: tool: Vitest with a throwaway Postgres in Docker | scope: All route handlers including role and ownership matrix, rate limiter per-IP separation

**e2e_testing**: tool: Playwright | key_flows: ["browse map and list","denied geolocation fallback","add library","add and remove book","report and moderate","magic-link login with confirm page","account deletion"]

**test_data_strategy**: Seed of 20 libraries at a fixed point; tile requests mocked

**ci_test_gates**: Lint, typecheck, unit, integration and e2e must pass to merge

**manual_qa_checklist**
- Phone Safari map
- Phone Chrome map
- Magic link in a real inbox
- Screen reader on list view

### Risks
- Flaky map tests **Mitigation:** Mock tiles and wait on pin elements

### Agent notes
- Tests run only against the Docker database, never staging or prod.

## 11. Error Logging & Debugging

Sentry with scrubbing, rotated structured logs, and an external uptime monitor.

### Decisions
- **External uptime check on /api/health**: A crashed server cannot report its own outage.

### Implementation

**client_error_tool**: @sentry/nextjs browser SDK, tunneled via /monitoring

**server_error_tool**: @sentry/nextjs with instrumentation.ts

**log_levels**: error, warn, info, debug via LOG_LEVEL

**log_destinations**: stdout to pm2 with pm2-logrotate (10 MB, retain 14, compress)

**alert_thresholds**: Email on more than 10 errors in 5 minutes; uptime monitor pages on 2 failed checks; alert at 80 percent of EMAIL_DAILY_CAP

**debug_output_strategy**: LOG_LEVEL=debug on staging only

**source_maps**: Uploaded in CI with SENTRY_AUTH_TOKEN

**correlation_ids**: x-request-id set in middleware

### Risks
- PII reaches Sentry **Mitigation:** sendDefaultPii false, beforeSend scrubber

### Agent notes
- One logger module; no console.log (lint rule).

## 12. Code Health & Structural Integrity

The Code Auditor blocks work through a required failing CI check backed by concrete thresholds.

### Decisions
- **Standards encoded as lint rules and CI thresholds**: Prose standards are not enforced.

### Implementation

**during_build_standards**
- no-explicit-any
- max-lines 300
- no-console
- zod validation on every handler
- authz helper on every mutating handler

**static_analysis_tools**
- tool: ESLint | purpose: Style and bugs | config: next/core-web-vitals, no-explicit-any, max-lines 300, no-console
- tool: tsc --noEmit | purpose: Types | config: strict
- tool: Prettier | purpose: Formatting | config: default

**ci_gates**
- check: lint | blocking: true
- check: typecheck | blocking: true
- check: tests and coverage 80% on lib/ | blocking: true
- check: madge zero cycles | blocking: true
- check: jscpd under 3% | blocking: true
- check: npm audit high | blocking: true
- check: knip | blocking: false

**post_build_audit**: architectural_hotspot_method: madge cycle report plus files over 300 lines | dependency_audit_tool: npm audit | duplicate_detection_tool: jscpd | dead_code_tool: knip (blocking after Phase 2)

**auditor_agent_handoff**: The Code Auditor runs the gate suite, posts a pass/fail report listing violated gates, and sets the required 'code-audit' CI status; the Lead Builder must clear every blocking item before the phase closes.

### Risks
- Gates slow merging **Mitigation:** CI budget of 6 minutes

### Agent notes
- The Auditor does not edit source; its only write is the status check and the report.

## 13. Deployment & CI/CD

GitHub Actions builds a standalone bundle and deploys with health-gated rollback.

### Decisions
- **Standalone bundle, release folders, pm2 startOrReload from ecosystem.config.js**: Avoids the pm2 cwd and symlink trap.

### Implementation

**ci_tool**: GitHub Actions

**cd_strategy**: main deploys staging; prod tag requires environment approval

**build_steps**
- npm ci
- lint
- typecheck
- unit and integration tests
- playwright e2e
- next build (standalone)
- upload source maps

**deploy_steps**
- rsync standalone bundle to releases/<ts>
- run migrations with the migration role
- start new release with explicit cwd via pm2 startOrReload
- poll /api/health up to 60 s
- switch the current symlink only after a passing check

**rollback_strategy**: On a failed health check, keep the previous release running and revert the symlink; migrations are expand/contract so the old release stays valid

**health_checks**: /api/health checks the database; external uptime monitor

**zero_downtime_approach**: pm2 cluster mode with 2 instances and startOrReload

### Risks
- Deploy SSH key theft **Mitigation:** Dedicated deploy user, pinned host key, sudo limited to pm2 reload

### Agent notes
- Keep five releases; prune older ones.

## 14. Cost & Budget Estimate

About 45 dollars a month with Postmark paid from launch and staging on the same server.

### Decisions
- **Pay for Postmark from launch**: Login must not hit a free-tier cap.

### Implementation

**infrastructure**
- item: VPS (prod and staging) | monthly_cost: $14
- item: Object storage (photos and backups) | monthly_cost: $2
- item: Domain | monthly_cost: $1
- item: Uptime monitor | monthly_cost: $0 (free tier)

**third_party_apis**
- service: Postmark | free_tier: 100 emails/month | paid_tier: $15/month (used)
- service: Sentry | free_tier: 5k errors/month (used) | paid_tier: $26/month
- service: Cloudflare Turnstile | free_tier: free | paid_tier: n/a

**total_monthly_estimate_low**: $32

**total_monthly_estimate_high**: $58

**cost_optimization_notes**: High end assumes the Sentry paid plan or a keyed tile provider.

### Risks
- Cost creep from tile provider switch **Mitigation:** Budget alert and TILE_URL swap only if OSM blocks

### Agent notes
- Review spend monthly.

## 15. Timeline & Phasing

Seven part-time weeks in four phases, with photos and moderation scheduled explicitly.

### Decisions
- **Reports and admin ship in Contribute**: Moderation exists before public launch.

### Implementation

**phases**
- name: Foundation | duration: 1 week | deliverables: ["Repo, CI gates, database roles, seed data, Postmark approval request, DNS records"] | dependencies: []
- name: Browse | duration: 2 weeks | deliverables: ["Map, list view, library pages, geolocation fallback"] | dependencies: ["Foundation"]
- name: Contribute | duration: 3 weeks | deliverables: ["Auth, add and edit library, books, check-ins, photo pipeline, reports, admin tools, account deletion"] | dependencies: ["Browse"]
- name: Launch | duration: 1 week | deliverables: ["Hardening, backup restore drill, uptime monitor, privacy page, go live"] | dependencies: ["Contribute"]

**total_estimate**: 7 weeks

**critical_path_items**
- Postmark approval and DNS
- Photo upload pipeline
- Authorization test suite

### Risks
- Contribute phase overruns **Mitigation:** Photos are optional and can slip to v1.1

### Agent notes
- Browse passes when its Playwright flows and the 3 second pin-render check pass on the seed.

## 16. AI Subagent Team

Five agents with directory ownership and a concrete audit blocking mechanism.

### Decisions
- **One owner per directory, migration PR before feature PR**: Prevents overlapping edits.

### Implementation

**agents**
- role: Lead Builder | specialty: Next.js and TypeScript | model: Sonnet | tools_needed: ["Read","Write","Edit","Bash","git"] | responsibilities: ["Owns app/ and components/","Feature PRs"] | handoff_protocol: Opens a PR per feature after the migration PR merges
- role: Data Engineer | specialty: Postgres and Drizzle | model: Sonnet | tools_needed: ["Read","Write","Edit","Bash","git"] | responsibilities: ["Owns db/","Schema, migrations, seed"] | handoff_protocol: Opens and merges migration PRs first
- role: QA Engineer | specialty: Vitest and Playwright | model: Sonnet | tools_needed: ["Read","Write","Edit","Bash","git"] | responsibilities: ["Owns tests/","Writes tests from the acceptance criteria before the feature PR"] | handoff_protocol: Reports failing criteria in the PR
- role: DevOps / Release | specialty: nginx, pm2, GitHub Actions, DNS, email | model: Sonnet | tools_needed: ["Read","Write","Edit","Bash","gh"] | responsibilities: ["Owns infra/ and .github/","Deploy, backups, monitoring, SPF/DKIM/DMARC"] | handoff_protocol: Confirms staging deploy before each phase review
- role: Code Auditor | specialty: Code health | model: Opus | tools_needed: ["Read","Grep","Bash","gh (status checks only)"] | responsibilities: ["Owns section 12","Sets the required code-audit status"] | handoff_protocol: Posts a pass/fail report after each phase

### Risks
- Agents block each other **Mitigation:** Fixed order per feature: migration, tests, feature, audit

### Agent notes
- No agent edits another agent's directory without a PR to its owner.

## Open items the council still had at finalize time

Scores: {"security":6,"architecture":6,"implementer":6,"ops":6,"completeness":6}

Much stronger than round 1: every round-1 contradiction is resolved, and scores held at 6 only because the council now finds second-order problems. Remaining HIGH items are mechanical: the deploy rollback does not match how pm2 behaves, the CSP blocks the Turnstile widget the login depends on, Next.js standalone output omits static assets, a single 4 GB box is exposed to out-of-memory kills, and the Code Auditor still cannot enforce anything because nobody defines which identity posts its status. Note: the Ops persona ran three web searches this round (OSM tile policy, Postmark approval, Next.js standalone with pm2); in round 1 it reported it did not search.

- **[high] deployment**: startOrReload replaces live workers before the health check, so the previous release cannot keep running; standalone output omits .next/static and public; old and new build hashes break in-flight clients. Fix: Start the candidate on a spare port, health-check it, then flip nginx upstream; copy .next/static and public into the bundle; keep one prior release's static files.
- **[high] security**: CSP blocks the Turnstile script and iframe; no style-src; data: missing for markers; Referrer-Policy must not be no-referrer or OSM blocks tiles. Fix: Allow challenges.cloudflare.com in script-src, frame-src and connect-src; add style-src and data: img; set Referrer-Policy strict-origin-when-cross-origin; add an e2e login test under the production CSP.
- **[high] security**: Postgres-backed limiter writes on every read; mail-bombing a victim email can burn the global email cap and lock out real logins. Fix: Reads limited in nginx only; per-recipient, per-IP and reserved-quota email budgets; Turnstile on every anonymous send.
- **[high] server**: Prod, staging, Postgres, sharp and builds share 4 GB; a memory spike can OOM-kill Postgres. Fix: Build only in CI, add swap, systemd memory limits, sharp concurrency 1, consider staging on demand or a separate small instance.
- **[high] code_health**: Auditor cannot enforce: no identity defined for the status check, authz and zod rules are prose, section reference off by one. Fix: Make code-audit a GitHub Actions job under branch protection; add a lint test that every app/api handler uses requireRole and zod.
- **[high] database**: No audit log table; account deletion semantics and foreign keys are undefined; no geocode cache. Fix: Add audit_log, geocode_cache, uploads tables; define deletion as a transaction with tombstoned email hash and ON DELETE SET NULL.
- **[high] navigation**: List tab has no route, confirm and edit routes and admin endpoints are missing. Fix: Add ?view=list, /login/confirm, /library/[id]/edit and the admin API endpoints.
- **[high] agent_team**: lib/ and middleware have no owner; tests-before-feature deadlocks against required green CI. Fix: Lead Builder owns lib/ and middleware; tests land in the same PR as the feature.
