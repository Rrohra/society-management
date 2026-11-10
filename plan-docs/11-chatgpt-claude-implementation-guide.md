# 11. ChatGPT Plus + Claude Implementation Guide

## Purpose

Build the foundation with ChatGPT Plus in small sessions, then use Claude only for the parts that need deep reasoning, sending Claude just the files it needs. This guide references the other documents instead of repeating them; when a prompt says “attach”, paste or upload that excerpt.

**Current repo state (verify before starting):** Gradle build; project `society-platform`; one module `tools:devdb`; Spring Boot 4.1.2; Java 21; PostgreSQL (embedded PostgreSQL for dev/tests); no application code.

Docs 03 and 10 mention a Maven layout; **this guide uses the Gradle layout already in the repo**. Fix those mentions when convenient.

Do not assume ChatGPT limits. Rules used here instead: one milestone per chat, one concern per prompt, ask for files in batches (see “If the reply is cut off”), and stop at the end of each milestone with green tests.

Adapted to this project. Anything marked **safety** must exist and be tested in the ChatGPT version; Claude then deepens it. Nothing safety-related is deferred.

## 11.1 Responsibility matrix

| Area | ChatGPT builds (baseline, tested) | Claude enhances later |
|---|---|---|
| Build, modules, config | Gradle modules, Spring profiles, Vite app, lint/test scripts | — |
| Tenancy (docs 10.4) | `TenantResolver`, `TenantContext`, fail-closed `RoutingDataSource`, per-tenant Flyway, **safety:** cross-tenant and “no context → exception” tests | Pool budgeting, job context propagation, canary/fleet `societyctl migrate`, review for leaks |
| Identity | Invitation, login/logout, opaque session in DB, CSRF, rate limit hook, password hashing, problem+json errors | MFA, step-up, lockout/anti-enumeration timing, session revocation edge cases, forgot/reset hardening |
| Authorization | Interface, role + unit-scope checks for simple rules, **safety:** matrix test generated from doc 02 | Record-level policy edge cases (joint owners, ended occupancy, last-admin rule) |
| Audit | Append-only `AuditRecorder`, DB grants (no UPDATE/DELETE) | Tamper-evidence, retention |
| Property, vehicles | Entities, DTOs, CRUD, validation, pagination, ETag/If-Match | Occupancy history rules, duplicate-registration edge cases |
| Notices, notifications | CRUD, publish/archive, in-app notifications, email outbox table | Outbox worker, retry/backoff, preferences logic |
| Documents | Metadata CRUD, `DocumentStorage` port + local-disk adapter, download-link endpoint | Scan pipeline (ClamAV), type sniffing, quarantine, short-lived signed links, object-storage adapter |
| Billing | Components, policy table, cycle **preview/issue for flat-rate**, bills, offline payment, receipt PDF, collection report | Per-sq-ft/component charges, late fee/interest, partial allocation, advances, adjustments, rounding |
| Payments | `PaymentGateway` port, explicit `NotConfigured` adapter returning 501, payment-intent and webhook-inbox tables, webhook endpoint that stores the raw body and returns 501 until verified | Real adapter, signature verification, idempotency, reconciliation, refunds |
| Cards, map | Card entity, QR via ZXing, minimal verify endpoint, SVG map + hotspot CRUD | Revocation/privacy review, expiry rules |
| Frontend | Shell, router, API client, theme tokens, feature flags, slot registry, forms/tables/dialogs, screens for baseline CRUD | Complex screens (billing preview, payment status polling), a11y fixes, performance |
| Tests | Unit + slice + Testcontainers-free tests using one happy-path and the **safety** cases per module | Edge-case, concurrency and property-based tests for the enhanced feature |

### Rules for ChatGPT code (so Claude can work module by module)

1. Package by feature per doc 03 §3.3. Other modules are called only through their public `Service` / port interface.
2. Deferred behaviour must fail loudly: throw mapped to HTTP 501 `problem+json` with `code=NOT_IMPLEMENTED`. No fake success, no silent fallback, no hidden mock beans outside tests.
3. Every deferred item gets a comment and a row in the module's `LIMITATIONS.md` (max 15 lines).
4. Names: tables/columns per doc 04; endpoints per doc 03 §3.6.
5. Dependencies: only Spring Boot starters (web, security, data-jpa, validation, actuator), Flyway, PostgreSQL driver, springdoc, ZXing, OpenPDF, ArchUnit (tests). Frontend: React, React Router, TanStack Query, Vite, Vitest, Testing Library. Ask before adding anything else.

## 11.2 Roadmap (dependency order)

Each milestone is one ChatGPT session, ends with green verification, and yields a handoff note (11.4). Roughly maps to phases 1a–4 of doc 08 §8.6.

| # | Milestone | Prerequisite | Claude pass afterwards |
|---|---|---|---|
| M0 | Skeleton and conventions | — | — |
| M1 | Tenancy core | M0 | **C1** (recommended before M2) |
| M2 | Society schema, migrations, tenant config | M1 | — |
| M3 | Errors, audit, session auth baseline | M2 | **C2** |
| M4 | Authorization policy + permission-matrix test | M3 | **C3** |
| M5 | Property CRUD | M4 | — |
| M6 | Frontend shell, theme, flags, slots, auth screens | M3 | — |
| M7 | Vehicles API and property/vehicle screens | M5, M6 | — |
| M8 | Notices and notifications | M5 | **C6** |
| M9 | Documents baseline | M5 | **C5** |
| M10 | Billing baseline | M5 | **C4** |
| M11 | Payments scaffold | M10 | **C7** |
| M12 | Cards and map | M5, M6 | — |
| M13 | Fleet and ops basics | M2 | **C8** |

## 11.3 Standard context block and ChatGPT prompts

### Reusable preamble (paste at the top of every ChatGPT session)

```text
You are implementing one milestone of a multi-society residential-management platform.

Stack: Java 21, Spring Boot 4.1.2, Gradle (Groovy DSL), PostgreSQL via Flyway, React + TypeScript + Vite.
Architecture: modular monolith, package-by-feature (web -> application -> domain -> infra), database per society, tenant resolved from hostname, REST /api/v1, errors as RFC 9457 problem+json, money as string decimals, UUID ids, ETag/If-Match on updates.

Rules:
- Work only on this milestone. Do not touch other modules except where listed.
- Deliver files in batches of at most two files; say which batch you are on and what remains. Show full file contents, not diffs; for changed files show the full changed method/class.
- Keep it simple: no new dependencies beyond those I list, no extra abstractions.
- Unfinished behaviour must throw NotImplementedYetException (HTTP 501, code NOT_IMPLEMENTED). Never return fake success, never add silent fallbacks, never add mock beans outside tests.
- Mark deferred items: DEFERRED(id): what. Add them to the module's LIMITATIONS.md.
- All safety checks (authorization, tenant isolation, input validation, CSRF) are in scope and must have tests.
- End with: (1) the list of files created/changed, (2) the commands I should run, (3) a handoff summary of at most 12 lines.
- If anything below is ambiguous in a way that changes the design, ask before writing code; otherwise proceed.
```

Attach per milestone only the excerpts named under **Attach**. Previous handoff note goes after the preamble.

### M0 — Skeleton and conventions

- **Objective:** buildable multi-module project with shared conventions. **Prereq:** none.
- **Create:** `libs/tenancy/build.gradle`, `apps/society-app/build.gradle`, settings.gradle update, `frontend/` (Vite + TS + Vitest), `NotImplementedYetException`, `ProblemDetail` handler, ArchUnit module-rule test, README run section.
- **Attach:** `/settings.gradle`, `build.gradle`, doc 03 §3.3 (module list) and §3.5 (error format).

```text
Milestone M0. Extend the existing Gradle build (attached) with modules libs:tenancy and apps:society-app. Keep the existing tools:devdb module untouched. Create a frontend/Vite+React+TS app with Vitest.

In society-app create: SocietyApplication; platform package with NotImplementedYetException, a @RestControllerAdvice producing RFC 9457 problem+json (fields: type, title, status, detail, instance, traceId, errors[]) for 400/401/403/404/409/412/422/429/500/501 with no stack traces; GET /api/v1/ping returning {"status":"ok"}; an ArchUnit test asserting no module accesses another module's infra package, and that no class in src/main is named Mock or Fake. Add package placeholders (package-info.java) for identity, property, vehicles, documents, notices, notifications, billing, payments, cards, buildingmap, audit, tenancy, tenantconfig, platform.
```

- **Acceptance:** `./gradlew build` green; `/api/v1/ping` works; 501 test passes; `npm run build` and `npm test` green in frontend.
- **Handoff:** module list, package root, how errors are raised, build commands.

### M1 — Tenancy core

- **Objective:** hostname → tenant resolution, request context, routing datasource, per-tenant migration entry points. **Prereq:** M0.
- **Create:** `TenantResolver`, `TenantContext`, `TenantFilter`, fail-closed `RoutingDataSource`, pool registry, `TenantMigrator`, tests for unknown host, suspended host, no context, two-tenant routing, context clear after request.
- **Attach:** doc 10 §10.2–10.4; doc 05 §5.11.
- **Acceptance:** all six tests pass; no default DataSource bean exists.
- **Verify:** `./gradlew :libs:tenancy:test`.
- **Handoff:** public types, how to bind context in jobs (`TenantContext.runAs(tenant, fn)`), DEFERRED ids.

### M2 — Society schema, migrations, tenant config

- **Objective:** society DB schema via Flyway for all tables, per-tenant migrate on startup in dev, tenant-config endpoint. **Prereq:** M1.
- **Create:** `db/migration/` grouped per module; `TenantMigrator`; `tenantconfig` module (`TenantConfig` record, JSON schema validation, `GET /api/v1/public/tenant-config` public subset only).
- **Attach:** doc 04 §4.1 conventions and §4.3 table list; doc 10 §10.5.

```text
Milestone M2. Create the Flyway migrations for the society database exactly following the attached table definitions and conventions (UUID ids, row version, timestamps in UTC, society id column, constraints and indexes as specified). Implement TenantMigrator (runs Flyway for one tenant using its own connection) and a dev-profile runner that migrates all ACTIVE tenants at startup. Implement the tenantconfig module and the public endpoint, returning only branding, locale, features, navigation, extension keys (no secrets) cached.

Do NOT add triggers or logic beyond the specified constraints. Audit table: grant INSERT/SELECT only to the app role. Tests: migrations apply on an empty DB; re-run is a no-op; config with an unknown key is rejected; the public endpoint never returns fields marked private.
```

- **Acceptance:** migrations apply to two tenants; schema matches doc 04; config validation test passes.
- **Verify:** `./gradlew :apps:society-app:test`.
- **Handoff:** migration file list, config keys, how to add a migration.

### M3 — Audit and session authentication baseline

- **Objective:** invite + set password → login + session + logout. **Prereq:** M2.
- **Attach:** doc 05 §5.1–5.2; doc 03 §3.6 Auth rows.

```text
Milestone M3. Implement session-based auth per the attached security rules, inside identity and audit. Implement: POST /auth/invitations/{token}/accept, POST /auth/login, POST /auth/logout, GET /me. Opaque random session id (store only a hash), stored in the tenant DB, idle and absolute timeouts from config, rotated on login. Cookie: __Host- prefix, HttpOnly, Secure, SameSite=Lax. CSRF via X-XSRF-TOKEN on state-changing requests. Generic login failure message and equal work for unknown email. Invitation token hashed at rest, single use, expiring. Every login success/failure, logout and invitation accept writes an audit event.

MFA, password forgot/reset: throw NotImplementedYetException (DEFERRED ids M3-MFA, M3-RESET); the login response must NOT claim MFA is satisfied for ADMIN roles: those users cannot log in until MFA exists (return 403 code=MFA_REQUIRED).
```

- **Tests:** wrong password and unknown user give identical response; session from tenant A rejected on tenant B; CSRF rejected; expired invitation rejected; session fixation id changes.
- **Acceptance:** all tests pass; no secret or token logged.
- **Verify:** `./gradlew :apps:society-app:test --tests '*identity*'`.
- **Handoff:** filter order, session table, DEFERRED ids, test fixtures for creating users.

### M4 — Authorization policy and permission-matrix test

- **Attach:** doc 02 §2.1–2.2; doc 05 §5.3.

```text
Milestone M4. Implement AccessPolicy exactly per the attached matrix and rules. Roles ADMIN, TREASURER are explicit assignments; OWNER/CO_OWNER/TENANT/RESIDENT_MEMBER come from UnitAccessQuery (interface only; use an in-memory test double inside tests only). Effective permissions are the union, each evaluated in its own scope. Unauthorized or not-visible resources return 404; authenticated-but-forbidden actions return 403. Generate permission-matrix.csv (role, action, scope, expected) from the matrix and write a Parameterized test that fails if any row is unmatched. List any matrix cell you found ambiguous instead of guessing.
```

- **Acceptance:** every matrix row has a test; ambiguities listed for you to decide.
- **Verify:** `./gradlew test --tests '*PermissionMatrix*'`.

### M5 — Property CRUD

- **Attach:** doc 04 Property tables; doc 03 §3.5 and §3.6 rows for Units/Ownership/Tenancy.

```text
Milestone M5. Implement the Property module per the attached tables and endpoints. Use DTOs only at the boundary. Validate inputs (lengths, enums, date ranges, no overlapping active occupancy for the same person+unit beyond what the schema allows). Writes require If-Match -> 412 on stale. Every write calls AuditRecorder. Joint ownership: shares must be validated to sum to 100 where specified in the schema; otherwise 422. Anything about occupancy history beyond simple end-dating: DEFERRED(P-HISTORY). Tests: CRUD happy paths; 422 on invalid; 412 on stale; scoped read (owner sees only own units, others 404); tenant isolation test using two tenants.
```

- **Verify:** `./gradlew :apps:society-app:test --tests '*property*'`.

### M6 — Frontend shell, theme, flags, slots, auth screens

- **Objective:** one build, themed and flag-driven. **Prereq:** M3 (M2 for config endpoint).
- **Create:** tenant-config bootstrap, CSS variables from tokens, route/menu from flags, typed slot registry, login and accept-invitation screens, route guards, `problem+json` mapping, Button/Field/Table/Dialog/Toast primitives.
- **Attach:** doc 10 §10.6, doc 03 §3.4, OpenAPI JSON or `/me`, `/auth/*`, tenant-config response examples.

```text
Milestone M6. Build the front-end shell per the attached design. No UI library, no Redux. Fetch GET /public/tenant-config before rendering protected routes; apply CSS variables and feature/navigation config; build the typed slot registry. Implement login, invitation acceptance, route guards, problem+json error mapping, and accessible Button, Field, Table, Dialog and Toast primitives. Use React Router, TanStack Query, Vitest and Testing Library. Include one sample extension and a test proving it overrides a slot. Client-side permission checks only hide UI; the API remains authoritative.
```

- **Acceptance:** config controls theme and navigation; a disabled feature has no route/menu entry; unauthenticated routes redirect to login; sample slot override test passes.
- **Verify:** `cd frontend && npm run lint && npm test && npm run build`.

### M7 — Vehicles API and property/vehicle screens

```text
Milestone M7. Backend: vehicles module per attached schema/endpoints (unit-scoped CRUD, admin lookup by registration number, uniqueness -> 422 DUPLICATE, audit). Frontend: screens for units list/detail, owners, occupancies, vehicles using the existing primitives, TanStack Query, optimistic concurrency (send If-Match; show a reload prompt on 412). Respect feature flags. Tests on both sides, including 404-for-not-visible handling.
```

- **Verify:** `./gradlew :apps:society-app:test --tests '*vehicle*'` and `cd frontend && npm test && npm run build`.

### M8 — Notices and notifications

```text
Implement notices (draft/publish/archive, audience) and in-app notifications (list, read, read-all), and create an email-outbox row in the same transaction as the notification. Do NOT send email: the sender throws NotImplementedYetException; outbox rows stay PENDING (DEFERRED(N-OUTBOX)). Publishing a notice creates notifications for the audience. Tests: audience scoping, publish only by ADMIN, outbox row created and read affects only the caller. Add matching front-end list/detail screens.
```

- **Verify:** `./gradlew :apps:society-app:test --tests '*notice*' --tests '*notification*'` and `cd frontend && npm test`.

### M9 — Documents baseline

```text
Milestone M9. Implement documents: multipart upload with metadata, size limit from config, allow-list of content types, random storage key under tenants/{tenant_key}/..., DocumentStorage port with a local-disk adapter (dev only, path from config), visibility checks, download-link endpoint. Scan status is stored as PENDING and the download endpoint REFUSES (409 code=SCAN_PENDING) any document whose scan status is not CLEAN. The scanner port throws NotImplementedYetException (DEFERRED(D-SCAN)) so uploads remain PENDING: do not mark anything CLEAN. Original filename is never used as a path. Tests: oversize, bad type, path traversal name, cross-tenant key, visibility, download refused while PENDING.
```

- **Verify:** `./gradlew :apps:society-app:test --tests '*document*'`.

### M10 — Billing baseline

```text
Milestone M10. Implement billing for FLAT_RATE charge components only. Components and policy CRUD (TREASURER), cycle preview and issue (Idempotency-Key required; re-issuing the same cycle returns the original result, never duplicates), bills with derived status, unit ledger, offline payment recording (TREASURER, audited, allocated oldest-first to a single bill; overpayment is rejected with 422 for now), receipt PDF via OpenPDF, collection and outstanding reports (+csv). Use integer minor units or NUMERIC(14,2) per the schema; round half-up at one defined place.

Everything else — PER_SQFT / component charges, late fee, interest, partial allocation across bills, advances, adjustments, refunds — throw NotImplementedYetException with ids B-CHARGES, B-LATEFEE, B-ALLOC, B-ADJ. Billing is the only writer of bills/ledger. Tests: preview equals issue; idempotent issue; no duplicate bill per unit+cycle (concurrent test with two threads); ledger sums equal bill totals; Admin cannot edit financial rows.
```

### M11 — Payments scaffold

```text
Milestone M11. Implement the Payments scaffold, not a gateway. Port PaymentGateway {createIntent, verifyWebhook, refund}. The only adapter is NotConfiguredGateway: every method throws NotImplementedYetException (DEFERRED(PAY-ADAPTER)). POST /bills/{id}/payment-intents requires Idempotency-Key, checks authorization and bill state, persists a payment_intent row in state CREATED_UNSENT and then returns 501 — it must never return a redirect URL. POST /webhooks/payments/{provider}: no session, CSRF-exempt, stores the raw body, headers and received_at in the webhook inbox with verified=false, and returns 501 (never 200). Unique constraints per attached rules. Payments is the only writer of payments. Tests: 501 behaviours, idempotency-key required, inbox row stored with verified=false, unauthorized user gets 404.
```

### M12 — Cards and map

```text
Milestone M12. Cards: issue (ADMIN), revoke, GET /me/card with QR (ZXing) of a random token stored hashed; GET /public/card-verify?t= returns only {valid, displayName initials, expiry} — minimal; rate limited; revoked/expired -> valid=false. Map: SVG asset upload via documents-style checks (no scripts: reject SVG containing <script>, on* attributes, external references), hotspot CRUD linking to units, GET /map, GET /map/units/{id} with a text alternative. Front-end screens for both. Tests: token never stored in clear, revoked card, malicious SVG rejected.
```

### M13 — Fleet and ops basics

```text
Milestone M13. Create tools:societyctl (Java, picocli or plain args) with: onboard --key --host --admin-email (creates DB, runs migrations, inserts registry row, creates first ADMIN invitation), migrate --tenant key | --all, list, suspend, resume. Idempotent and re-runnable; failing migration stops --all and exits non-zero. Canary ordering, backup/restore, offboarding: DEFERRED(OPS-FLEET). Tests against tools:devdb with two tenants.
```

### If a reply is cut off

- If a reply stops mid-way, send: “Continue from batch N. Output only the remaining files, same rules.” Never ask it to “redo everything”.
- If the model drifts (new dependencies, fake success), reply: “Revert to the preamble rules. List violations, fix them, change nothing else.”

## 11.4 Project-context and session-handoff template

Keep a single file `docs/HANDOFF.md` (not committed to history beyond latest; overwrite each time), max ~40 lines. Paste it after the preamble in the next ChatGPT session, and reuse it as the base for Claude handoffs.

```md
# Handoff after <Mx> (date)

Built & verified: <modules/endpoints>; verify cmd: <cmd> -> green on <date>
Key types/ports (path -> purpose):
Conventions in force: <errors, ids, auth, tenant binding>
Deferred (id -> file -> current behaviour):
Known issues:
Next milestone: <Mx+1> — needs from this one:
Per-module LIMITATIONS.md (< 15 lines): id | current behaviour | limitation | intended enhancement | test that pins current behaviour.
```

These files are the main thing Claude should read, together with the module's source.

## 11.5 Claude enhancement tasks

### Claude handoff template (fill in, paste, attach only listed files)

```text
Task <Cx>: <one-line goal>
Implemented & verified: <from HANDOFF.md, only this module>; tests green via <cmd>
Files to read (attached): <= 10 paths: module source, its LIMITATIONS.md, its tests, ports it uses
Interfaces I may call but not change: <signatures>
Known limitations / deferred ids to implement: <ids from LIMITATIONS.md>
Enhancement requested: <precise behaviour, inputs, outputs, rules, doc § refs>
Must remain unchanged: <public endpoints, DTO fields, DB columns, error codes, existing test expectations>
Acceptance: <numbered, testable>; new tests in <path>; run <cmd>
Output: changed files only, full contents for changed methods, plus any new migration (expand-only).
```

How to keep it cheap: attach the module directory plus its ports, not the repo; attach doc sections by number, not whole docs; ask Claude for diffs of changed files only; start a fresh session per task: if Claude needs another file it will say which.

If you work from the repo in Claude Code, simply name the module and the doc sections and let it read only those.

### Targeted prompts

**C1 — Tenancy review and hardening** (after M1). Attach `libs/tenancy`, doc 10 §10.4, doc 05 §5.11.

```text
Review and harden libs/tenancy against the attached design. Find any path where a connection could be obtained without a bound context, or where context leaks between requests/threads. Implement: context-propagating executor decorator for jobs, pool budget check (sum of max pools + admin <= 70% of max_connections), secrets resolved via envelope-encrypted store replacing DEFERRED(T-SECRETS). Keep public types unchanged. Add tests for each leak scenario.
```

**C2 — Auth hardening** (after M3): implement DEFERRED M3-MFA (TOTP + recovery codes, mandatory for ADMIN/TREASURER, step-up for role changes) and M3-RESET (forgot/reset without account enumeration, token single-use, sessions revoked on reset), lockout/backoff. Keep existing login contract; replace the MFA_REQUIRED block with the real flow. Tests include timing-equal responses.

**C3 — Authorization edge cases** (after M4/M5): resolve the ambiguities listed in `LIMITATIONS.md` using my decisions; implement joint-owner, ended occupancy, and last-admin rules and make the matrix test cover them. Do not change endpoint contracts.

**C4 — Billing engine** (after M10): implement B-CHARGES, B-LATEFEE, B-ALLOC, B-ADJ per policy decisions. Strategy beans selected by tenant config (no tenant names in code). Money rounding in one place; property-based ledger and concurrency tests; existing endpoints/DTOs unchanged; additive fields only; expand-only migration.

**C5 — Document scanning** (after M9): implement D-SCAN: ClamAV adapter, async scan job with tenant context, content sniffing (not extension), quarantine, CLEAN only after a positive scan, scan failure leaves PENDING with alert, short-lived signed download links. Keep download refusal while not CLEAN.

**C6 — Notification outbox worker** (after M8): implement N-OUTBOX: SKIP LOCKED worker iterating tenants with explicit context binding, retry with backoff, dead-letter state, email adapter via port, respecting notification preferences. Tests for crash mid-send (no duplicate).

**C7 — Payments** (after M11): implement PAY-ADAPTER for `<gateway>`: createIntent, webhook signature verification with raw body, idempotent webhook processing (inbox unique key), reconciliation job, refunds with approval, per-tenant credentials. Test duplicate webhook, out-of-order, amount mismatch, late success after expiry, refund over amount. Webhook returns 200 only after durable verified storage.

**C8 — Fleet operations** (after M13): implement OPS-FLEET: canary-first migrate with stop-on-failure, schema_version in registry, N-1 compatibility check at first tenant use, per-tenant backup/restore commands, offboard/export.

**C9 — Performance and accessibility** (before go-live): target `<metric and number>`. Measure first, propose the smallest change that meets it, implement, show before/after. Do not change contracts.

## 11.6 Final integration and verification checklist

Run once all milestones and the Claude passes you need are done. Tick each with evidence (command output or screenshot).

**Build and tests**

- [ ] `./gradlew build` green; `cd frontend && npm ci && npm run lint && npm test && npm run build` green.
- [ ] ArchUnit: no cross-module infra access, no `Mock` / `Fake` in `src/main`, no tenant names in business code.
- [ ] Every `NotImplementedYetException` is listed in a `LIMITATIONS.md` and the feature is hidden by a feature flag or documented as unavailable.
- [ ] Two-tenant suite: no data, session, cache, storage key or job leaks.
- [ ] Permission-matrix test covers every row of doc 02 §2.2.
- [ ] ADMIN/TREASURER cannot log in without MFA; CSRF enforced; cookies are HttpOnly, Secure and `__Host-`.
- [ ] Documents never downloadable before CLEAN; webhook returns 200 only after verified storage.
- [ ] No secrets, tokens or personal data in logs; 5xx responses carry only trace id.

**Safety**

- [ ] Two-tenant suite: no data, session, cache, storage key or job leaks.
- [ ] Permission-matrix test covers every row of doc 02 §2.2.
- [ ] ADMIN/TREASURER cannot log in without MFA; CSRF enforced; cookies are HttpOnly, Secure and `__Host-`.
- [ ] Documents never downloadable before CLEAN; webhook returns 200 only after verified storage.
- [ ] No secrets, tokens or personal data in logs; 5xx responses carry only trace id.

**Functional (doc 09 acceptance criteria)**

- [ ] Walk the journeys in doc 02 §2.3 on two onboarded tenants with different themes.
- [ ] Billing: issue cycle twice → one set of bills; offline payment → receipt PDF; reports match ledger.
- [ ] Payment sandbox end-to-end including duplicate and out-of-order webhooks (after C7).
- [ ] `societyctl onboard` on a fresh tenant works and is re-runnable; migrate stops on failure.

**Non-functional**

- [ ] Performance targets doc 08 §8.2 measured; accessibility checks doc 08 §8.3 (axe + keyboard pass).
- [ ] Backup/restore drill for one tenant (doc 08 §8.4).
- [ ] Dependency list equals the allow-list in 11.1 plus anything explicitly approved.

**Documentation**

- [ ] Every module has an accurate README; status updated from “Design” phase.
- [ ] Docs 03/10 repo-layout mentions (Maven) corrected to Gradle.

## 11.7 Decisions needed before specific milestones

Only these materially change the plan; defaults in doc 01 §1.3 apply until answered.

| Before | Question |
|---|---|
| M10 | Billing policy (charge basis, late fee, partial payments, advances) |
| M11 / C7 | Payment gateway and methods |
| M12 | Card format, expiry, who may verify |
| C2 | Confirm TOTP (authenticator app) as the MFA method |
