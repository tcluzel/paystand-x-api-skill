# PSX API — Implementation Lessons (from real ERP integrations)

Generalized from real Paystand X API integration projects (Sage X3, Sage 100, and multi-entity US/Canada rollouts) run with ERP integration partners. Customer names, ticket numbers, personal contacts, and internal URLs removed — safe to share externally. These are **project / architecture / onboarding lessons**, distinct from the endpoint-level gotchas in `gotchas.md`.

## Hard architectural constraint: PSX is NOT a second AR/reconciliation layer

PSX API is built for ERPs where Paystand has **no native integration**. Do **not** run PSX as a **second AR / reconciliation layer** alongside a Paystand native ERP integration (NetSuite, Sage Intacct, Business Central, Acumatica) against the same accounting system — overlapping ownership of receivables/payments causes reconciliation conflicts, broken transfer reporting, duplicate payment records, and ERP accounting inconsistencies. If a merchant already has a native integration, that integration stays responsible for ERP-side receivables, payment application, and reconciliation.

**The nuance (embedded / ISV / vertical-software cases):** an external application using **Paystand at the point of payment** — while the native integration still handles the ERP-side outcome — is a *different* motion from “PSX as a second connector,” and is evaluated via the **Paystand Checkout/API pattern**, not by bolting PSX on. As of 2026-09 this coexistence pattern is a hypothesis under SE/Implementation validation, not a blanket-supported design; scope it with Paystand before promising it.

**Three connectivity categories (don't conflate them):**
- **Native ERP integration** — Paystand built and productized the ERP connector.
- **Paystand X** — an external system interacts **directly** with Paystand (this API).
- **iPaaS / EDI** — moves data between systems; explicitly **not** what PSX is.

**Four coexistence rules** any multi-system design (e.g. POS/order app + Paystand + ERP) must satisfy:
1. **One system owns each record/action** (order, invoice, payment record, customer record) — no shared ownership.
2. **Systems share durable transaction IDs** so records can be matched across systems.
3. **The ERP record (e.g. the order/invoice) exists at the point reconciliation requires it** — sequence creation so the ERP side is ready before payment must reconcile.
4. **Retries / webhooks must not create duplicate ERP records or payments** — make every create/apply idempotent (tie to the durable IDs above).

Before pursuing coexistence, get a yes/no/depends from Paystand on: coexistence, reconciliation back to the right ERP order/invoice, required sequencing and ID hand-offs, record ownership, order/inventory path, and the boundaries that would force everything onto a single integration path.

## Onboarding sequence (what actually gates go-live)

Before any API code runs, these non-code items are the usual blockers:

1. **Merchant application(s) & underwriting approval** — one application **per entity/subsidiary** in scope (e.g. a US entity and a Canadian entity are separate applications and separate approvals).
2. **Paystand X technical onboarding Jotform** — there are **two different Jotforms**: a general company-onboarding form and a separate **Paystand X technical onboarding form**. Teams routinely complete the first and miss the second; the missing PSX form is the #1 silent setup blocker. Make sure the PSX-specific form is submitted.
3. **Branding materials** — logo (PNG, ≤150×150px) and primary/secondary colors in hex.
4. **Sender-domain DNS (email auth via Mandrill)** — to send payer emails from the merchant's own domain (e.g. `ar@merchant.com`), add these DNS records, then Paystand verifies:
   - `CNAME  mte1._domainkey.<domain>  →  dkim1.mandrillapp.com`
   - `CNAME  mte2._domainkey.<domain>  →  dkim2.mandrillapp.com`
   - `TXT    mandrill_verify           →  mandrill_verify.<token>`
   - `TXT    @                         →  v=spf1 include:spf.mandrillapp.com ?all`
   Domain won't verify until the `mandrill_verify` TXT is present. Without domain auth, emails send from `noreply@paystand.com` / `support@paystand.com` and are more likely to hit spam.

## Credential handoff at kickoff

A PSX API integration needs credentials flowing **both ways**:

**Paystand → integrator:** `X-CUSTOMER-ID` (merchant customer ID), `client_id`, `client_secret` — found on the dashboard **Integrations** page **per instance** (see multi-entity note below). No facilitator/platform token needed.

**Merchant → integrator** (on-prem ERP install, e.g. Sage X3): a **temporary ERP/application-server user** for the integration developer with app-server access + create/delete on the ERP folder, **removed once installation is complete**.

**Multi-entity merchants:** each entity/instance (e.g. US and Canada) has its **own** `client_id`/`client_secret` on its own Integrations page, and API access may need enabling **separately per instance**. If credentials don't appear for one entity, that's a per-instance enablement request to Paystand, not a bug.

## Environment split (dashboard + API)

- **Sandbox = `.co`** (`dashboard.paystand.co`, API `api.paystand.co/v3`)
- **Production = `.com`** (`dashboard.paystand.com`, API `api.paystand.com/v3`)
- Sandbox and production dashboards/credentials are separate. Do all UAT in `.co`.

## Data migration of saved payment methods (bank/ACH)

When migrating existing customers' saved bank details, the file is exchanged securely and has strict field rules:

- **Transport:** SFTP with the integrator's **IP address(es) whitelisted**, and file **encrypted with Paystand's public PGP key**.
- **Required fields:** bank account holder **name** (mandatory), holder email (optional), holder **billing address incl. ZIP** (ZIP mandatory), bank **account number** (mandatory), bank **routing/ABA** (mandatory), **Customer ID** (mandatory).
- **Format account & routing numbers as TEXT** to preserve leading zeros — a spreadsheet turning `000123456789` into `123456789` will break the bank record.

## Auto-generated Journal Entries: required ERP fields must be DEFAULTED in the ERP

When Paystand auto-creates entries in the ERP (e.g. **Bank Transfer Journal Entries** from deposit/transfer data), the posting is **fully automated — no human fills any field.** If the ERP **requires** a field on those entries (e.g. a Department / analytical dimension), it must be **defaulted inside the ERP**, because the headless posting won't prompt for or inject it.

- **The defaulting must have a source.** As the ERP consultant put it: *“it has to come from somewhere in setup to know when to default it in, else it must be part of the import.”* Two working patterns: (a) default the dimension **on the GL account** so the posting inherits it from the referenced account, or (b) include the value in the import data.
- **Sage X3 specifics (example):** set the dimension default via **Default Dimensions** (`GESCDE`, code `GACCD`). Apply it to **both TEST and PROD** endpoints — not just TEST. Validate by reversing and reposting a test JE.
- Paystand and the AR-side partner **cannot make ERP setup changes** — route these to the ERP maintainer/administrator (often a different firm than the AR integration partner).

## Required customer data to sync

- **Every customer must have an associated email contact** to sync/import successfully. Missing email is a common import failure.
- Multiple emails allowed (first = primary, rest = secondary); enable “Auto CC to secondary emails” if secondaries should receive reminders.

## Known functional gaps surfaced in real integrations

- **No payer self-service login portal by default.** Payers pay via emailed payment/statement links, not a username/password login. A standalone customer login experience is a **separate product (Network Portal)** that must be added — don't promise a payer portal out of the box with PSX.
- **Partial payments** must be explicitly enabled/confirmed (Integrations → Payment Experience) — verify per payment experience.
- **PDF invoice pull** requires the ERP-side **file path** to the invoice PDFs to be identified and configured (an IT task on the merchant side).
- **Credit memos** surface in the payment experience, but the API does not APPLY them — apply in the ERP and re-sync (see `gotchas.md`).

## Scope SSO (and anything non-standard) explicitly and early

**Paystand does not provide native SSO** for these API integrations. If the merchant expects SSO, scope it **separately with the integration partner** as tailored work — it is not out-of-the-box. Surface non-standard auth/SSO requirements during the **initial scoping call**, not after records are loaded; discovering them late pushes go-live.

## Realistic timelines

- On-prem ERP connector build: partner estimate ~**2–3 weeks** for install + configuration, clock starting at **access/credential handoff**, not contract signing.
- Validate merchant-driven go-live dates against connector build time **and** underwriting/DNS/onboarding-form lead time.

## Roles to identify on an ERP-partner integration

- **AR platform (Paystand):** provides API, credentials, AR-record setup, guidance; **cannot modify the merchant's ERP.**
- **Integration partner / ERP developer:** builds and installs the connector, maps fields, aligns ERP config with the Paystand AR setup.
- **ERP maintainer/administrator (often a different firm):** owns ERP setup changes (dimension defaults, required-field config, GL account setup, print/file-path services). Confirm who this is early — required-field and JE issues route to them, not Paystand.

## Note on meeting recordings

Much partner/implementation decision-making lives only in **recorded calls** (shared via Gong or the partner's file storage), not in email or the API docs. If a scoping/training decision isn't in written docs, check the call recording before assuming it wasn't decided.
