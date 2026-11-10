# 1. Overview, Scope, Assumptions

## 1.1 Executive summary

A single-society web application with one login and role-based dashboards. It replaces spreadsheets, paper registers and chat-group notices with: unit/owner/tenant records; vehicle registry; document vault; notices and notifications; maintenance billing with hosted online payments; membership cards and a simple building map.

Key design choices:

| Decision | Choice | Rationale / trade-off |
|---|---|---|
| Architecture | Modular monolith, REST/JSON | One deployable, one DB transaction scope (important for money). Microservices add ops cost a volunteer committee cannot sustain. Modules have enforced boundaries so extraction is possible later. |
| Frontend | SPA (React + TS), served as static files | Simple CDN/reverse-proxy hosting. SSR not needed (authenticated app, no SEO). |
| Auth | Server-side session cookie (opaque ID, stored in PostgreSQL) | Easy revocation, no tokens in JS storage. JWT rejected: revocation difficulty, no benefit for a single first-party SPA. |
| Payments | Hosted gateway checkout (redirect/hosted fields) + signed webhooks | App never sees card data (keeps PCI scope minimal; likely SAQ-A, to be confirmed with the gateway). |
| Documents | Metadata in PostgreSQL, bytes in private object storage (or private disk volume for MVP) | Keeps DB small; access only via authorised, short-lived downloads. |
| Multi-tenancy | **Multi-society from day one:** database per society, hybrid shared/dedicated runtime, one themed front end with per-society slot overrides, per-society payment gateway accounts, CLI onboarding | Strong isolation and easy per-society backup/delete; costs fleet migrations and tenant-aware plumbing. Full design in [doc 10](10-multi-tenancy.md). |

---

## 1.2 Assumptions (explicit)

| ID | Assumption | Impact if wrong |
|---|---|---|
| A1 | Up to ~10 societies in the first 1-2 years, each about 50-500 units with 1-3 committee admins, onboarded manually by the platform operator. | 20+ societies need a console UI, connection-pool/fleet review and possibly separate clusters. |
| A1b | Each society has its own database, its own users, and its own payment-gateway account; the platform never holds funds. | An aggregator model is regulated and needs a different design. |
| A2 | Operated by volunteers; no 24/7 on-call. Target availability 99.5% (≈ 3.6 h/month downtime). | Higher targets need redundancy and cost more. |
| A3 | Currency is a single currency (configurable), amounts stored as integer minor units or NUMERIC(14,2). | Multi-currency out of scope. |
| A4 | "Society card" = membership/resident ID card, not a bank card. | Entirely different feature if wrong. |
| A5 | Users are invited by admin; no public self-registration. | Public sign-up adds abuse/verification burden. |
| A6 | Email is the primary identity and notification channel; SMS/WhatsApp deferred. | Phone-first users need an OTP design. |
| A7 | A "property owner" is a legal owner of one or more units. Ownership grants rights over that unit's data only, never admin rights. | See open question Q3. |
| A8 | Payment gateway supports hosted checkout, signed webhooks, idempotency keys and refunds via API. | Gateway selection must verify. |
| A9 | Content in English, single time zone (society's local zone, stored as UTC). | i18n deferred; keep strings externalised. |
| A10 | Real resident data is never used outside production. | See [environments](07-deployment-environments.md). |

---

## 1.3 Open questions (business decisions NOT made by this document)

| ID | Question | Why it matters | Default used meanwhile |
|---|---|---|---|
| Q1 | Jurisdiction (country/state)? | Data-protection law (e.g., DPDP Act, GDPR), payment regulation, society/co-operative law, retention periods, tax/receipt format. | None assumed; controls are generic and flagged for legal review. |
| Q2 | Society size and growth; number of buildings/wings? | Sizing, pagination, hosting tier. | ≤ 500 units. |
| Q3 | What does "society owner" mean: flat owner, builder/promoter, committee chair, or system owner? | Drives role model and who can administer. | Treated as flat owner with no admin power; "society administrator" is a separate committee-granted role. |
| Q4 | Membership card: physical, digital, or both? Expiry? Who may verify (guards, other residents)? Photo needed? | Privacy of verification, data collected. | Digital card with QR, no photo, expiry optional. |
| Q5 | Maintenance policies: charge basis (flat rate / per sq ft / per component), billing frequency, grace period, late-fee formula, interest, whether advances carry forward, partial payment allocation order, who can waive fees? | Core billing logic. | Configurable; defaults in [data model](04-data-model.md) section 4.6. |
| Q6 | Which payment gateway and methods (UPI, cards, net banking)? Gateway fees pass to residents? | Webhook format, refund API, cost. | Provider-agnostic port/adapter. |
| Q7 | Must tenants pay maintenance, or only owners? Who is billed (owner vs occupant)? | Bill-to party model. | Bill goes to unit; owner is liable; tenant may pay on owner's behalf if enabled. |
| Q8 | Are vehicle details visible to all residents, or only admin/security? | Privacy. | Admin and the unit's own members only. |
| Q9 | Document retention periods (financial: statutory, KYC-type docs: avoid)? | Retention job config. | Configurable per category. |
| Q10 | Is a security guard/staff role needed (vehicle lookup, card verification)? | Additional role. | Deferred; verification via limited-scope endpoint. |
| Q11 | Hosting preference/data-residency requirement? | Provider selection. | Single-region VPS or small managed platform in-country. |
| Q12 | Is an existing data set to be migrated (spreadsheets)? | Import tooling. | Admin CSV import, scheduled with data import in the pilot phase (Phase 5). |

---

## 1.4 Scope

### MVP (Phase 1-2)
- Unit, owner, and tenant records with history tracking
- Vehicle registry and parking allocation mapping
- Document vault with category and visibility isolation
- Notices and notification dispatching
- Maintenance billing cycle generation and ledger accounting
- Hosted online payment with webhook verification
- Digital membership card (QR code verification)
- Basic building map (SVG/image text alternative)

### Deferred

| Enhancement | Reason deferred |
|---|---|
| SMS/WhatsApp notifications | Per-message cost, provider approvals |
| Self-service society signup, billing of societies, platform console UI | CLI onboarding suffices for 10 societies |
| Digital membership QR with offline guard app | Needs Q4/Q10 answers |
| Complaint/helpdesk, amenity booking, visitor management | Out of requested scope |
| Accounting ledger integration (Tally etc.), GST/tax invoicing | Jurisdiction-dependent |
| Single sign-on across societies | Per-society accounts chosen (Q15) |
| Native mobile apps | Responsive PWA is enough initially |
| Interactive map | SVG with hotspots suffices |
| Auto-debit/recurring mandates | Regulatory and gateway complexity |

---

## 1.5 Multiple societies (day-one requirement)

Superseding the earlier single-society assumption. Everything below is specified in [10 Multi-society design](10-multi-tenancy.md):

1. **Database per society:** tenant resolved from hostname; routing DataSource that fails closed.
2. **Hybrid runtime:** shared app group by default, dedicated instance of the same artifact when needed.
3. **Platform control plane:** separate app + DB with tenant registry and `societyctl` onboarding CLI.
4. **One React build:** per-society theme tokens, feature flags, and typed component-override slots.
5. **Per-society accounts:** secrets (envelope encryption), gateway credentials, storage prefix, email identity.
6. **Fleet migrations:** mandatory expand/contract, per-tenant backup/restore, tenant-labelled monitoring.

