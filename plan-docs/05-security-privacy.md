# 5. Security, Privacy and Threat Controls

> Compliance note: this document specifies technical controls. It does not claim compliance with any law or standard. Privacy law, payment regulation and record-retention duties depend on jurisdiction (open question Q1) and need legal review.

## 5.1 Authentication and account lifecycle

| Area | Control |
|---|---|
| Invitation | Admin creates invite for a known person+email. Token: 256-bit random, stored as SHA-256 hash, single-use, expires 72 h, bound to email. Admin can revoke. No public sign-up. |
| Registration | Invitee sets password (min 12 chars, check against breached-password list e.g., offline k-anonymity list/zxcvbn; no composition rules, no forced periodic rotation). |
| Password hashing | Argon2id (Spring, tuned to ~250 ms on production hardware) or bcrypt cost ≥ 12 as fallback; unique salts; optional server-side pepper in secret store. |
| Login | Generic error messages ("Email or password incorrect"); constant-time compare; dummy hash for unknown email to avoid timing leaks. |
| Brute force | Per-account + per-IP throttling (progressive delay, 5 failures → 15 min lock); 429 responses; alerts on spikes. Lock notice emailed to user. |
| MFA | Mandatory for Admin and Treasurer (TOTP, RFC 6238; recovery codes hashed). Optional for others. Step-up MFA for role changes, refunds, approvals, bulk export. WebAuthn/passkeys as phase-3 enhancement. |
| Password reset | Generic response regardless of email existence; token 256-bit, hashed, 30 min, single use; all sessions revoked on reset; notification email sent; privileged accounts must also pass MFA. |
| Deactivation | Admin deactivates; sessions revoked immediately, role assignments end-dated, invitations voided; person record kept. Reactivation audited. |
| Email change | Requires re-authentication, confirmation link to new address and notice to old. |

## 5.2 Sessions

| Control | Setting |
|---|---|
| Mechanism | Server-side session (opaque 256-bit ID, hashed in `user_sessions`). |
| Cookie | `Secure`, `HttpOnly`, `SameSite=Lax` (Strict for admin paths if no cross-site flow). |
| Timeouts | Idle 30 min (privileged 15 min), absolute 12 h; "remember me" not offered for privileged roles. |
| Fixation | New session ID on login and on privilege/MFA step-up. |
| Logout / revocation | Server deletes session; "log out other devices"; admin can revoke a user's sessions. |
| Concurrent sessions | Listed under Profile → Security. |

## 5.3 Authorization (backend-enforced)

Three layers, all on the server:

1. **URL/method rules** in Spring Security (authenticated vs public; webhook and verification endpoints explicit).
2. **Method security** (`@PreAuthorize`) calling a central `AccessPolicy` (role + scope), deny by default.
3. **Record-level checks** in the application layer: every query for unit-scoped data is parameterised by the caller's permitted unit IDs (from `unit_ownerships` / `occupancies`)—e.g. `findBillById(id, allowedUnitIds)` rather than fetch-then-check. Out-of-scope record is 404.

Additional protections against cross-unit access (IDOR):

- UUID identifiers (no sequential IDs), but never rely on obscurity.
- DTO-based response; field-level filtering per role (e.g., co-owner phone hidden from tenants).
- Download endpoints re-check authorization on every request.
- Automated negative tests: for every endpoint, a user from unit B requests unit A's resource and must get 403/404 (see testing doc).
- Optional defence in depth: PostgreSQL Row-Level Security by `society_id`, required before multi-society consolidation.

## 5.4 Web security controls

| Threat | Controls |
|---|---|
| CSRF | Spring Security CSRF with cookie-to-header token for SPA (`CookieCsrfTokenRepository` + `XorCsrfTokenRequestAttributeHandler`), `SameSite=Lax`, check `Origin` / `Referer` on state-changing requests; webhook endpoint exempt but signature-verified. |
| XSS | React escapes by default; ban `dangerouslySetInnerHTML` (lint rule); notice bodies stored as Markdown and rendered through a sanitiser (DOMPurify / server-side OWASP Java HTML Sanitizer allowlist); strict CSP (below); no inline scripts. |
| SQL injection | Parameterised queries only (JPA/JDBC named params); no string concatenation; sort/filter fields from allowlists; DB role without DDL. |
| Brute force / abuse | Rate limits at reverse proxy and app (login, reset, invitation accept, verify-card, download links). |
| Clickjacking | `frame-ancestors 'none'`. |
| SSRF | No user-controlled outbound URLs; gateway/email hosts fixed in config. |
| Mass assignment | Explicit request DTOs; no binding to entities. |
| Open redirect | Redirect targets from an allowlist (payment return URLs fixed server-side). |
| Dependency risk | Dependabot/Renovate, OWASP Dependency-Check or Trivy, lock files, SBOM; patch SLA: critical 7 days. |

### Transport and headers

| Header / setting | Value |
|---|---|
| HTTPS | TLS 1.2+ (prefer 1.3), automatic certs (Let's Encrypt), HTTP→HTTPS redirect. |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` (preload only after confirming). |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; form-action 'self' <gateway host>; base-uri 'none'`. |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` (or `no-referrer`) |
| `Permissions-Policy` | Disable camera, mic, geolocation unless needed. |
| `Cache-Control` | `no-store` on authenticated API and download responses. |
| CORS | Prefer same-origin (SPA and API behind one host) so no CORS needed. If split, allow exactly one origin, specific methods/headers, credentials only for that origin; never `*` with credentials. |

## 5.5 Data protection

| Area | Control |
|---|---|
| In transit | TLS to browser; TLS to DB (`sslmode=verify-full` if DB is on a separate host); TLS for SMTP and gateway. |
| At rest | Full-disk/volume encryption for DB and storage; object-storage server-side encryption; encrypted backups (age/GPG or provider KMS). |
| Field-level | TOTP secrets and gateway credentials encrypted with an application key (AES-GCM), key stored outside DB; consider encrypting phone numbers only if risk assessment justifies it. |
| Secrets | Never in Git or images. Env vars injected from a secret store (cloud secret manager, or `sops`/`age`-encrypted file on a small VPS, root-only permissions). Rotation procedure documented; separate secrets per environment. Pre-commit/CI secret scanning (`gitleaks`). |
| DB least privilege | Runtime vs migrator accounts (see data model 4.8); no superuser from the app; DB reachable only from the app host/private network. |
| Logs | Structured JSON; masking filter for email, phone, tokens, auth headers, vehicle numbers; never log request bodies of auth/payment endpoints; trace ID for correlation; logs retained 30–90 days. |

## 5.6 Document upload security

| Control | Specification |
|---|---|
| Allowed types | PDF, JPEG, PNG (add DOCX/XLSX only if needed). Allowlist by **detected** content type (Apache Tika/magic bytes), not extension or client MIME. |
| Size | 10 MB per file default; per-user quota. |
| Filename | Never used as storage key: `society/{yyyy}/{uuid}`. Original name stored only as metadata and sanitised for display. |
| Content hardening | Reject encrypted/macro-enabled office files; for images, re-encode strips metadata/EXIF (also removes GPS); PDF checked for active content (JavaScript/launch actions) or simply served as attachment. |
| Malware scan | ClamAV (free) via `clamd`; status `PENDING_SCAN` → `AVAILABLE` / `QUARANTINED`; files not downloadable until clean; signatures updated daily; scan failure stays pending and alerts admins. |
| Storage | Private bucket/volume, no public URL, outside web root. |
| Download | Backend authorizes, then streams with `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff`, `no-store`; or issues short-lived (< 60 s) pre-signed URL to private object storage after authorization. |
| Access | See permission matrix; private documents invisible to admins unless explicitly shared or an audited break-glass action with reason. |
| Retention | `retain_until` per category; archive vs purge job; legal-hold flag. |
| Identity documents | **Not collected** (Aadhaar, passport, SSN, etc.) unless a justified requirement is documented and approved. If ever needed: separate category, explicit consent, encryption, shortest retention, masked display. |
| Abuse | Rate limit uploads; zip/decompression-bomb guard (no archive types in the allowlist). |

## 5.7 Payment security

- Hosted checkout or gateway-provided hosted fields only; app never receives PAN/CVV (target: simplest PCI self-assessment level, confirm with gateway).
- Webhook: HMAC signature check on raw body, timestamp tolerance, IP allowlist if provided, then server-to-server fetch to confirm.
- Credentials per environment (test keys in dev/staging; live keys only in prod).
- Return-URL pages show provisional status only; **payment is completed only when verified server-side** (see sequence diagram).
- Refunds require maker-checker and MFA step-up.

## 5.8 Audit trail

| Category | Events audited |
|---|---|
| Access | Login success/fail, MFA enable/disable, password reset, session revocation, role grant/revoke, user deactivate/reactivate. |
| Property | Ownership/occupancy create/end, unit edits, owner contact edits. |
| Documents | Upload, scan result, visibility change, download of private/sensitive docs, archive/purge, break-glass access. |
| Financial | Bill issue, adjustment request/approve, payment initiated/verified/failed, offline payment recorded, allocation, refund, receipt void/reissue, policy change. |
| Admin | Settings changes, exports. |

Record: who (user, role), what (action, entity, before/after minimal diff), when, from where (hashed IP), trace ID. Append-only table, DB-level protection, no UI to edit/delete; periodic export to cold storage; hash chaining (each row stores hash of previous) is an optional tamper-evidence enhancement.

## 5.9 Privacy and data minimisation

| Principle | Application |
|---|---|
| Collect only what is needed | Name, email, optional phone, unit relationship. No DOB, ID numbers, photos unless justified. |
| Purpose limitation | Vehicle data for parking management; payment data for billing only. |
| Visibility | Residents see only their unit; owner/co-owner contacts shared per policy; vehicle lookup limited to admin. |
| Public QR verification | Returns only `VALID/EXPIRED/REVOKED`, society name, and masked holder name (e.g., "R. S****"). No unit, phone, or email. Token is random, unguessable, rate-limited, revocable. No sequential card numbers in QR. |
| Retention and deletion | Per section 4.7: data subject requests (access/correction/erasure) handled by admin workflow with audit; erasure = anonymisation where law requires financial records be kept. |
| Notices to users | Privacy notice and consent text shown at first login; versioned acceptance stored. |
| Processors | List of subprocessors (hosting, email, gateway) documented in privacy notice. |
| Breach readiness | Incident runbook: detect, contain, assess, notify per jurisdiction timelines (Q1). |

## 5.11 Tenant isolation (multi-society)

| Control | Specification |
|---|---|
| Tenant identification | From the request hostname only, via the registry. Client-supplied tenant headers/params are ignored. Unknown/suspended host returns a generic page. |
| Fail closed | No tenant context → no DataSource, error. No default database. Background jobs carry an explicit tenant key and bind context in `try/finally`. |
| Data boundary | Separate database, with separate runtime and migrator DB roles per society; role of society A has no `CONNECT` on society B. |
| Session scoping | Sessions stored in the society DB; host-only cookies; a cookie from one society cannot authenticate on another. |
| Secrets | Envelope encryption with per-tenant data keys; per-tenant gateway credentials and webhook secrets; webhook signature verified with the receiving tenant's secret. Offboarding destroys the data key. |
| Caches and storage | Tenant-keyed caches; storage prefix (or bucket/credentials) per tenant. |
| Operator access | Platform admins have separate login + mandatory MFA, no application path to resident data; all actions in platform audit log; break-glass DB access is time-boxed, approved, logged. |
| Logging | `tenant` label (key only) on all logs/metrics; PII masking unchanged. |
| Testing | Cross-tenant negative tests are part of CI (see testing doc). |
| Front-end overrides | Extensions run in the same origin as the app, so they are **trusted code** reviewed in the same repo (no runtime-uploaded or third-party extension code). CSP unchanged (no inline/eval). |

### 5.10 Threat model summary (STRIDE-oriented)

| Threat | Example | Mitigation |
|---|---|---|
| Spoofing | Credential stuffing | Breached-password check, throttling, MFA for privileged roles. |
| Tampering | Edit payment amount | Server-computed amounts, signature check, append-only ledger. |
| Repudiation | Treasurer denies refund | Audit log with actor, maker-checker. |
| Information disclosure | Owner views another flat's documents | Record-level authorization, 404 on non-visible. |
| Denial of service | Login/upload flooding | Rate limits, size limits, proxy timeouts. |
| Elevation of privilege | Owner calls admin endpoint | Deny-by-default, method security, tests per endpoint × role. |
| Supply chain | Dependency | Lockfiles, scanning, minimal dependencies, pinned base images. |
| Insider/admin | Browses private docs | Private docs not visible by default, break-glass with reason. |
| Cross-tenant | Shared cache/job routing, tenant binding, webhook sent to wrong host | Fail-closed routed DataSource, tenant-keyed caches, explicit job tenant binding, per-tenant secrets/signatures, cross-tenant tests. |
| Operator abuse | Control plane, no data path, audit, MFA, break-glass approval | Separate society data; negative tests. |
