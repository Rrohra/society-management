# 9. Acceptance Criteria

Format: Given/When/Then, condensed. IDs are referenced by tests. Each criterion is verified by automated tests unless marked manual (M).

## AC-TEN: Multi-society platform
- AC-TEN-1 The tenant is derived only from the hostname; unknown or suspended hosts show a generic page and expose no tenant data.
- AC-TEN-2 Each society's data is in its own database with its own credentials; the runtime role of one cannot connect to another (verified by test).
- AC-TEN-3 A session, UUID or file link from society A never grants access in society B (cross-tenant negative suite passes).
- AC-TEN-4 A request or background job without tenant context fails closed; jobs for different tenants never cross-write.
- AC-TEN-5 `societyctl onboard` provisions a new society (DB, schema, seed, config, keys, domain, first-admin invite, smoke test), is idempotent, and can resume after failure at any step.
- AC-TEN-6 Two societies with different theme, features and one component override run side by side from the same build; disabled features are absent from navigation and rejected by the API.
- AC-TEN-7 Every slot has a default implementation; each extension passes the contract, accessibility and size tests; extensions cannot import outside the public front-end API.
- AC-TEN-8 `societyctl migrate` upgrades a canary then all tenants; the app tolerates schema N-1; a lagging tenant shows maintenance without affecting others.
- AC-TEN-9 Each society's gateway credentials are stored encrypted and used only for that society; a webhook delivered to the wrong host fails signature verification.
- AC-TEN-10 A single society can be suspended, exported, restored from its own backup, moved to a dedicated runtime group, and offboarded with its files and data key destroyed without affecting other societies.
- AC-TEN-11 Platform admins cannot read resident data through any application endpoint; their actions appear in the platform audit log.
- AC-TEN-12 CI fails on tenant-name literals in business code and on config that violates the tenant-config schema.

## AC-AUTH: Identity, access, sessions
- AC-AUTH-1 Given an admin invites an email, when the invitee opens the link within 72 h, then they can set a password once; an expired link returns an error and nothing changes.
- AC-AUTH-2 Passwords < 12 chars or in the breached list are rejected with a clear message.
- AC-AUTH-3 After 5 failed logins the account locks for 15 min; response text is identical for unknown email.
- AC-AUTH-4 Admin and Treasurer cannot reach their dashboards without completing MFA; recovery codes work once.
- AC-AUTH-5 Password reset invalidates all sessions and the reset token; the response does not reveal whether the email exists.
- AC-AUTH-6 Deactivating a user ends their sessions immediately (next request returns 401).
- AC-AUTH-7 Session idle timeout and absolute timeout enforced; logout invalidates server-side.
- AC-AUTH-8 Owner can never access admin endpoints; role changes audited and require step-up MFA; last admin cannot be removed.

## AC-REC: Society and property records

- AC-REC-1 Admin can create buildings, floors, units; duplicate unit number on a floor is rejected.
- AC-REC-2 A unit can have multiple active owners whose shares total 100%; an owner can have multiple units.
- AC-REC-3 Ending an ownership preserves the old row with end date; past bills still show the owner at that time.
- AC-REC-4 Two overlapping active tenancies on the same unit are rejected unless flagged as co-occupants.
- AC-REC-5 Owner sees only own units; requesting another unit returns 404.

## AC-VEH: Vehicles

- AC-VEH-1 Registration numbers are normalised (case/space/hyphen-insensitive); a duplicate active registration returns 409 without revealing the other owner to non-admins.
- AC-VEH-2 Owner/resident can add/remove vehicles for own unit only; admin can search by registration.
- AC-VEH-3 Parking slot cannot be double-allocated for overlapping dates.

## AC-DOC: Documents

- AC-DOC-1 Only allowlisted types (by content) and configured size are accepted; others rejected with explanatory error.
- AC-DOC-2 EICAR test file ends `QUARANTINED`, is never downloadable, and admins are alerted.
- AC-DOC-3 A file is not downloadable until the scan passes; status visible to uploader.
- AC-DOC-4 Society-wide documents visible to all authenticated members; private documents visible only to permitted unit users; admin cannot see private docs without break-glass (audited).
- AC-DOC-5 Download links are authorised per request, expire < 60 s, and filenames cannot cause path traversal or header injection.
- AC-DOC-6 Archiving hides documents from default lists; retention job purges after `retain_until` with an audit event.

## AC-NOT: Notices and notifications

- AC-NOT-1 Admin publishes a notice to all/building/unit; only targeted users receive in-app notifications.
- AC-NOT-2 Unread count updates; marking read and "read all" works.
- AC-NOT-3 Email is sent only to users who enabled it; failures retry with backoff and don't block publishing.
- AC-NOT-4 Notice content with script tags renders inert (sanitised).

## AC-BILL: Maintenance billing

- AC-BILL-1 Treasurer can configure components, due dates; preview shows per-unit totals before issue.
- AC-BILL-2 Issuing a cycle creates exactly one bill per unit; repeating the action creates no duplicates.
- AC-BILL-3 Statuses (unpaid, partial, paid, overdue, adjusted) derive correctly from payments, adjustments and due date.
- AC-BILL-4 Late fee applied once per period per policy; capped, auditable; policy changes don't alter already-issued bills.
- AC-BILL-5 Partial payments allocate in the configured order; overpayment becomes advance credit and is applied to the next bill.
- AC-BILL-6 Adjustments/waivers require a second approver; bills are never edited in place.
- AC-BILL-7 Unit ledger balance equals the sum of bill/payment/adjustment entries; the nightly drift check reports zero difference on test data.

## AC-PAY: Payments and receipts

- AC-PAY-1 Offline payment (cash/cheque/transfer) is recorded by Treasurer with mode, reference, date; duplicate reference rejected; receipt generated; audit event written.
- AC-PAY-2 Pressing Pay twice results in a single intent/payment.
- AC-PAY-3 The app never receives or stores card data (verified by traffic and DB inspection (M)).
- AC-PAY-4 A bill is marked paid only after a verified webhook plus gateway confirmation with matching amount; the return-URL alone never completes a payment.
- AC-PAY-5 Duplicate and out-of-order webhooks produce no duplicate payments or negative balances.
- AC-PAY-6 Failed payments show a clear message and allow retry; failed attempts are retained.
- AC-PAY-7 Daily reconciliation lists mismatches/orphans; treasurer can resolve with audited action.
- AC-PAY-8 Refunds require request + different approver; refunded amounts never exceed paid; ledger shows reversal; receipt reflects credit.
- AC-PAY-9 Receipt PDF is downloadable by the payer/unit and treasurer, with gapless receipt numbers per society.
- AC-PAY-10 Collection report totals equal the sum of verified payments for the period and export to CSV.

## AC-CARD: Membership cards

- AC-CARD-1 Admin can issue, expire and revoke a card; only one active card per person per unit.
- AC-CARD-2 Holder sees digital card with QR; QR contains an opaque random token, not personal data.
- AC-CARD-3 Public verification returns only validity status, society name and masked name; revoked/expired cards report so immediately; endpoint is rate limited.

## AC-MAP: Building map

- AC-MAP-1 Authenticated users view the layout (wings, floors, common facilities) on mobile and desktop.
- AC-MAP-2 A text/list alternative provides the same information and is screen-reader accessible.
- AC-MAP-3 Layers marked restricted (e.g., utility rooms) are hidden from non-admins.

## AC-SEC: Cross-cutting security and privacy

- AC-SEC-1 Authorization matrix test suite passes for all endpoints/roles/scopes (IDOR cases return 403/404).
- AC-SEC-2 All required headers (HSTS, CSP, nosniff, etc.) present; CORS rejects unknown origins; cookies are Secure, HttpOnly, SameSite.
- AC-SEC-3 CSRF protection blocks state-changing requests without a token.
- AC-SEC-4 ZAP baseline shows no high findings; dependency scan shows no unaddressed high/critical.
- AC-SEC-5 Logs contain no passwords, tokens, full emails/phones or card/payment secrets (log scan test).
- AC-SEC-6 Audit events exist for all categories in 5.8 and cannot be updated or deleted via the application DB role.
- AC-SEC-7 No identity documents are collected; data export and anonymisation procedures work on synthetic data.

## AC-OPS: Operations , performance , accessibility

- AC-OPS-1 Dev and staging contain only synthetic data; prod startup fails if synthetic rows or dev secrets are detected.
- AC-OPS-2 Migrations run from empty and from the previous release; rollback procedure verified on staging.
- AC-OPS-3 Restore drill completes within RTO and passes integrity checks (M, quarterly).
- AC-OPS-4 Performance targets in 8.2 are met in k6/Lighthouse runs under the stated assumptions.
- AC-OPS-5 axe-core shows no serious violations; manual walkthrough of every major journey succeeds (M).
- AC-OPS-6 Monitoring alerts fire for health check failure, disk/cert expiry < 14 days, webhook failure spikes (verified by fault injection).
