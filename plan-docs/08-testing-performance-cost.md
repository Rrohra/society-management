# 8. Testing, Performance, Accessibility, Availability, Cost, Phasing

## 8.1 Testing strategy

| Level | Tools | Focus |
|---|---|---|
| Unit | JUnit 5, AssertJ; Vitest + Testing Library | Billing calculation, allocation, late fees, rounding, state machines, validators |
| Integration | Spring Boot Test + Testcontainers PostgreSQL, WireMock for gateway/email | Repositories, Flyway migrations, constraints, webhook flow, transaction boundaries |
| API contract | OpenAPI diff in CI | No unintended breaking changes |
| Architecture | ArchUnit / Spring Modulith verify | Module boundaries |
| End-to-end | Playwright (Chromium + mobile viewport) | Key journeys per role |
| Security | Authorization test matrix, OWASP ZAP baseline scan, dependency/container scan, secret scan | See below |
| Accessibility | axe-core in Playwright, manual keyboard + screen-reader pass | WCAG 2.2 AA target |
| Performance | k6 against staging with synthetic data | Targets in 8.2 |
| Backup/restore | Quarterly drill | Section 7.7 |

### Authorization tests (generated from the permission matrix)

- Parameterised test: for **every endpoint × every role × {own unit, other unit, no unit}** assert expected 200/403/404. Matrix file drives tests so a new endpoint without a row fails CI.
- Owner of unit A cannot read bills, documents, vehicles, tenants, receipts or notifications of unit B (IDOR, including via list filters, search, export, and download links).
- Owner with an ownership that has ended loses access immediately; former tenant likewise.
- Multi-role: owner who is also treasurer gets treasurer access society-wide but only own-unit private documents.
- Owner-is-not-admin: owner cannot reach admin endpoints; admin-by-ownership-assumption bug tests.
- Deactivated user session rejected; revoked session rejected; role removal effective on next request.
- Maker-checker: creator cannot approve own adjustment/refund.
- Last-admin removal blocked. MFA step-up enforced.
- Public endpoints leak nothing: card verification response schema contains only whitelisted fields; reset/login responses identical for unknown vs known email.
- CSRF token missing → 403. CORS from foreign origin rejected.
- File upload: wrong magic bytes, double extension, oversize, EICAR test file, path traversal filename, zero-byte file.

### Multi-society tests

- Integration fixtures start **two or more tenants** (separate Testcontainers databases) with different config.
- **Cross-tenant negative tests:** a session from tenant A sent to tenant B's host is rejected; the same user-ID/UUID in B is not visible; resource IDs from A return 404 on B; a spoofed `X-Tenant`/`Host` override header is ignored.
- Fail-closed: a request or job without tenant context cannot obtain a connection; unknown and suspended hosts return the generic page.
- Async: scheduled jobs, email outbox, scan callbacks and webhooks bind the right tenant; concurrency test interleaves jobs for two tenants and asserts no cross-writes.
- Cache and storage isolation: same keys in two tenants do not collide.
- Webhook for tenant A's payment replayed at tenant B's host fails signature verification.
- Migrations: canary-then-fleet run on N test tenants; app version N works on schema N-1 (expand/contract test); a tenant outside the supported schema range shows maintenance while others keep working.
- Config: invalid tenant config is rejected; `config upgrade` migrates old schema versions.
- Onboarding: `societyctl onboard` succeeds end-to-end on a clean environment, is idempotent on re-run, resumes after a forced failure at each step, and offboarding removes DB, storage and keys.
- Front end: every slot has a default; every extension passes the contract test (renders with fixture props, axe check, chunk size budget); lint rule blocks extension imports outside `core/public` and `design-system`; visual snapshot per theme.
- Static check: build fails on tenant-name literals in business code.

### Payment edge-case tests

| Scenario | Expected |
|---|---|
| Initiate twice with same Idempotency-Key | Same intent returned |
| Webhook delivered twice / ten times concurrently | One payment, one allocation, one receipt |
| Webhook with bad signature / stale timestamp | 400/401, nothing recorded as verified; logged |
| Webhook amount ≠ intent amount | `MISMATCH`, no allocation |
| `captured` webhook after `refunded` (out-of-order) | Final state consistent; no negative balances |
| Redirect success but no webhook | Stays `PROCESSING`; reconciliation completes |
| Failure then successful retry | One payment; failed attempt retained |
| Two users pay the same bill simultaneously | Both verified; second becomes advance/refund-flag; bill never over-allocated |
| Partial payment then remainder | Status transitions correct; receipt per payment |
| Payment arrives after bill adjusted/waived | Balance recalculated; excess handled by policy |
| Late fee job run twice | Single fee per period |
| Bill generation retried | `UQ(cycle, unit)` → no duplicates |
| Offline payment duplicate reference | Rejected with 409 |
| Refund > payment, double refund | Rejected |
| Concurrency | Property-based test: random sequences of events always yield ledger sum = bill balances |
| Gateway timeout during order creation | Intent expires, no orphan payment |
| Ledger drift job | Detects injected inconsistency |

Quality gates: ≥ 80% line coverage overall, ≥ 90% on `billing` and `payments`; mutation testing (PIT) on billing calculations; no high/critical vulnerabilities at release.

## 8.2 Performance targets

**Assumptions:** ≤ 500 units (~1,500 users), peak ≈ 50 concurrent users on bill due-date days (~5 req/s average, 20 req/s burst), data ≤ 5 years of history (≈ 100k bills/payments rows), server 2 vCPU / 4 GB RAM. Client reference: mid-range phone, **"Slow 4G"** profile (≈ 1.6 Mbps down, 150 ms RTT, Chrome DevTools/Lighthouse), warm DNS/TLS.

| Metric | Target |
|---|---|
| Initial JS (gzipped, login/dashboard route) | ≤ 150 KB; total transfer for first dashboard view ≤ 300 KB (excluding fonts/images from first view lazy-loaded) |
| Per-route lazy chunk | ≤ 50 KB gzipped |
| Fonts | System font stack or ≤ 1 subset WOFF2 (< 30 KB) |
| LCP (Slow 4G, mid phone) | ≤ 2.5 s (p75); on broadband < 1.5 s |
| INP | ≤ 200 ms |
| CLS | ≤ 0.1 |
| Lighthouse Performance / Accessibility | ≥ 90 / ≥ 95 |
| API latency (server side, excluding network) | Reads p95 < 200 ms, writes p95 < 400 ms, list endpoints with filters p95 < 300 ms; report generation (< 500 bills) < 5 s (async + download if longer) |
| Document download start | ≤ 1 s to first byte for files < 10 MB |
| Bill generation (500 units) | ≤ 60 s, background job with progress |
| Webhook processing | < 2 s from receipt to commit; always respond within gateway timeout |
| Error rate | ≤ 0.5% 5xx under target load |

Techniques: server-side pagination (default 20, max 100); indexed searches (trigram/b-tree); lazy-loaded routes and images; HTTP caching with hashed static asset names (`immutable`), gzip/brotli; ETag on reference data (units, components); avoid N+1 (entity graphs / DTO projections); HikariCP pool sized to CPU; in-memory Caffeine cache only for small reference data (roles/policies) with short TTL; no cache for authorization decisions longer than a request.

## 8.3 Accessibility and UX

| Area | Requirement |
|---|---|
| Standard | WCAG 2.2 AA target |
| Forms | Visible labels (not placeholder-only), `aria-describedby` errors, inline + summary error message, errors state what to do ("Enter a 10-digit phone number"), correct `autocomplete` / `inputmode`, no timeouts without warning |
| Keyboard | All actions reachable, visible focus, logical order, skip link, modals trap/return focus |
| Visuals | Contrast ≥ 4.5:1, base font ≥ 16 px, 44×44 px touch targets, respects `prefers-reduced-motion` and zoom to 200% |
| Semantics | Landmarks, headings hierarchy, tables with headers, status messages via `aria-live` (e.g., payment processing) |
| Map | SVG with `<title>`/`<desc>` plus an equivalent text/list view by wing → floor → unit |
| Language | Plain language, consistent terms; responsive mobile-first with bottom navigation on small screens |
| Animations | None required; loading skeletons only |

## 8.4 Availability, backup and operations targets

| Metric | Target |
|---|---|
| Availability | 99.5% monthly (excluding announced maintenance); Option A: 99.90% only with managed/redundant setup |
| RPO | < 24 h at MVP, ≤ 15 min after WAL archiving |
| RTO | ≤ 4 h |
| Restore drill | Quarterly, documented |
| Maintenance window | Planned, off peak (avoid billing due dates) |
| Monitoring alert response | Business hours acknowledgement |

## 8.5 Operating-cost considerations (relative; confirm with current provider pricing)

| Cost item | Driver | Notes |
|---|---|---|
| Shared platform group | Per additional society storage, backups, email, monitoring, share of compute; optional dedicated instance (a small extra VPS) | Keeps marginal cost low; decide who bears hosting cost (commercial model Q17) |
| Operations load | Fleet migrations, per-tenant restore drills, secret rotation, onboarding | Scripted with `societyctl`; time grows roughly linearly with societies |
| Compute | 2 vCPU / 4 GB typically enough | Largest fixed cost; consider reserved plan |
| Managed DB (optional) | Backup/PITR convenience | Often costs more than the VPS itself |
| Storage + off-site backup | Documents volume | Usually small; set per-user quotas |
| Domain + TLS | Domain yearly; TLS free | |
| Email | Transactional volume | Free tiers may suffice for a small society |
| Payment gateway | Fee per transaction, possible refund/chargeback fees | Decide who bears the fee (Q6); biggest variable cost |
| SMS/WhatsApp | Per message | Deferred; send only critical reminders |
| Monitoring/error tracking | Self-host vs hosted | Self-host adds ops load |
| People time | Patching, backups, restore drills, dependency updates | Hidden cost; budget ~ a few hours monthly |
| Compliance/legal review | One-off | Privacy notice, payment terms |

Cost-control tactics: single VPS, no Kubernetes/Redis/broker, quotas, scheduled jobs in-process, email not SMS, image optimisation on upload.

## 8.6 Phasing

| Phase | Duration (est., 1–2 devs) | Deliverables | Exit criteria |
|---|---:|---|---|
| 0. Discovery | 1–2 wks | Resolve open questions 01–Q12, gateway selection, wireframes | Signed-off decisions |
| 1a. Tenancy foundation | 3–4 wks | Control-plane DB and platform app, tenancy library (resolver, routing DataSource, fail-closed context, tenant-aware jobs), fleet Flyway, tenant config + schema validation, `societyctl` onboard/migrate/suspend, two synthetic tenants in CI | Cross-tenant test suite green; two tenants onboarded by script on staging |
| 1b. Foundation | 3–4 wks | Project skeleton, CI/CD, auth (invite, login, MFA, sessions), RBAC + record-level policy, audit, society/unit/owner/tenant records, synthetic seed, front-end shell with theme tokens, feature flags, slot registry and one sample extension | Authorization test matrix green; deployed to staging |
| 2. Core features | 4–6 wks | Vehicles, documents (+ scan), notices/notifications, building map | Feature acceptance criteria met |
| 3. Billing and payments | 5–7 wks | Charge components, cycles, bills, offline payment, receipts, reports, hosted payment + webhook reconciliation, adjustments/refunds | Payment edge-case suite green; sandbox end-to-end; treasurer UAT |
| 4. Cards and hardening | 3–4 wks | Membership cards + QR verify, per-tenant gateway credentials and webhook config, privacy export, fleet monitoring, pen-test/ZAP, performance tests (multi-tenant load), final audit, per-tenant backup/restore drill | Security gates met; restore drill passed |
| 5. Pilot and go-live | 2 wks | Limited pilot (committee + few residents), data import, training, go-live checklist | Go/no-go sign-off |
| Later | — | SMS/WhatsApp, guard role, passkeys, multi-society accounting export, helpdesk | Prioritised by feedback |
