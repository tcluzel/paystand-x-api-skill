---
name: psx-api
description: "Build an integration against the Paystand X (PSX) REST API — authentication, endpoints, webhooks, ERP sync patterns, sandbox testing, reconciliation, and known gotchas. Use when integrating an ERP or middleware/iPaaS with Paystand X, or when the user mentions Paystand X, PSX, the Paystand X API, receivables/payments sync, or Paystand webhooks."
license: MIT
metadata:
  version: "1.1.0"
---

# Paystand X (PSX) Public API

The Paystand X Public API is a **RESTful interface that syncs ERP accounts-receivable (AR) data with Paystand X**. It is ERP-agnostic (NetSuite, Sage X3, Business Central, Acumatica, QuickBooks, etc.) and system-to-system only — **there is no UI**. A developer (the merchant's team or an integration partner) builds a connector or uses middleware/iPaaS (Zapier, SmartConnect, Merge.dev, DataBlend) to move data. Paystand provides the API, docs, and guidance; the merchant/partner owns the connector.

**Canonical technical docs:** https://paystand-developers.netlify.app/docs/paystand-x-api (API version 1.1.0). This skill mirrors and annotates them with real-world gotchas; when they disagree, note it and prefer the behavior verified in `references/gotchas.md`.

## Base URLs

| Environment | Base URL |
|---|---|
| **Production** | `https://api.paystand.com/v3` |
| **Sandbox** | `https://api.paystand.co/v3`  (note `.co`, NOT `.com`) |

Mixing these up is a common cause of auth/404 failures. Sandbox = `.co`, production = `.com`.

## Authentication (OAuth2 client_credentials)

1. **Get a token:** `POST /v3/oauth/token` with JSON body:
   ```json
   { "grant_type": "client_credentials", "client_id": "<from Paystand>", "client_secret": "<from Paystand>", "scope": "auth" }
   ```
   `client_id` / `client_secret` come from **Dashboard → Integrations** (API Plan Settings must have API keys enabled). Response returns `access_token`, `token_type: Bearer`, `expires_in: 1209600` (**tokens valid 2 weeks**), and a `refresh_token` that is *reserved for future use* (do not rely on it — just re-request a token before expiry).

2. **Every authenticated call needs two headers:**
   ```
   Authorization: Bearer <access_token>
   X-CUSTOMER-ID: <merchant customer_id>     # from Dashboard → Integrations
   Accept: application/json
   ```
   The `X-CUSTOMER-ID` (merchant `customer_id`) is required on every request and is distinct from a payer customer ID (see ID chaining below).

- No facilitator/platform token needed — plain merchant API credentials are enough.
- `insufficient_scope` error → API keys not enabled on the plan (`show_api_keys`) or wrong `scope` (must be `auth`).

## Recommended sync order (ERP integration)

1. **Authenticate** → `POST /oauth/token`.
2. **Customers (payers)** → create/update via `POST /payerCustomers` / `PUT /payerCustomers/:id`. Store both Paystand `payerCustomer.id` and your `extCustomerId` (ERP key).
3. **Receivables (invoices)** → `POST /receivables/create`, linking `payerCustomerId` **or** `extCustomerId` (only one required; the customer must exist first). Poll changes with `GET /receivables` (filtered).
4. **Attachments** (optional) → `POST /receivables/:id/attachments` (multipart, one file per call, **20 MB each**). Send PDFs: when the merchant account enforces PDF-only attachments, the file *content* must be a PDF (the extension is not checked). The 3-attachment limit is enforced on files sent with Create Receivable, not on this endpoint.
5. **Payments & applications** → poll `GET /payments/all` and per-receivable transactions; subscribe to Payment / Receivable-Transaction webhooks when available.
6. **Fees** → merchant processing: `GET /fees` + Fee Events. Payer recoup/surcharge: `POST /feeSplits/splitFees` + `feeSplit` on Get Payment.
7. **Credit memos** → CRUD + cancel/activate.
8. **Refunds & disputes** → `GET /refunds/all`, `GET /disputes/all` + webhooks.
9. **Transfers (payouts)** → `GET /transfers` (filtered) for bank reconciliation.
10. **Withdrawals** → `GET /withdrawals` (filtered).

Before calendar-day jobs, optionally call `GET /customers/timezone` (Get Merchant Timezone).

## Webhooks vs polling (pick based on the merchant's stack)

- **Webhooks** — when the middleware/ERP has a public HTTPS endpoint. Configure in **Dashboard → Integrations → Webhook Event URLs**. See `references/webhooks.md`.
- **Polling** — for on-prem ERPs / partners without inbound HTTPS. Use the `f.querytype=by-query-filters` date-range pattern. See `references/endpoints.md` (“Incremental sync”).

## Voiding / deleting

There is **no hard delete** in the public API. Void an invoice with `POST /receivables/:id/cancel` (or update status to `cancelled`); void a credit memo with the cancel endpoint; remove a PDF with the delete-attachment route (removes the file, not the receivable). No public DELETE route for payer customers.

## When NOT to use PSX API (hard architectural constraint)

**PSX API must not run as a second AR / reconciliation layer on top of a Paystand NATIVE ERP integration** (NetSuite, Sage Intacct, Business Central, Acumatica). PSX is built for ERPs where Paystand has **no** native integration. Running PSX alongside a native integration against the same accounting system creates **overlapping ownership of receivables and payments** → reconciliation conflicts, broken transfer reporting, duplicate/mismatched payment records, and accounting inconsistencies in the ERP. An external/ISV app that only needs to *initiate payment* on top of a native integration is a different motion — evaluate the **Paystand Checkout/API pattern** (coexistence rules + open questions in `references/implementation-lessons.md`), not PSX as a second AR layer. (Design constraint per internal architecture review, 2026-09.)

## Critical known issues (read before you build)

- **Receivable fields are RENAMED between request and response.** You send `erpId / totalAmount / amountDue / postingDate / dueDate`; they come back as `extId / amount / amountPaid / date / dateDue` (`amountDue` is consumed: `amountPaid = totalAmount − amountDue`). **Match webhooks on `extId`; compute open balance as `amount − amountPaid`.** `erpId` can be changed later through Update Receivable, so keep it stable on your side — every webhook matches on it. (Full mapping in `references/gotchas.md`.)
- **The convenience (payer) fee is NOT a Fee event.** It lives on `payment.feeSplit` (`payerTotalFees` = fee, `subtotal` = invoice portion). Fee events carry only Paystand's *merchant* processing cost — and a fee is final only when `feeType` is `paystand`; a `delayed` fee has no usable amount, so don't post it.
- **Do NOT send `externalId` at the top level of a bank object** when adding a payer bank. The backend misreads it as a Stripe payment-method ID and tries to create a `StripeSetupIntent`, so bank creation fails. Put your reference in `meta.externalId` instead.
- **Guard against double-payment / double-counting.** (a) A payment applied in Paystand may lag syncing back to the ERP — sync status promptly (webhooks) so the same invoice isn't paid twice. (b) One checkout can emit many Receivable Transaction events sharing one `paymentId`; sum `amountApplied` — summing those *and* `payment.amount` double-counts.
- **Webhooks are at-least-once** — dedupe on event `id`; timestamps are authoritative, arrival order is not; return 2xx within 15 s.
- **`erpId` and `erpRef` are REQUIRED** on receivable create. Omitting either returns `400` with `detailCode: parameterMissing`, reported under the stored names (`extId` for `erpId`, `invoiceKey` for `erpRef`). There is no `invoiceId` alias.
- **Multi-entity merchants: enable API credentials per instance.** For a merchant with separate entities/instances (e.g. US + Canada), each instance exposes its **own** `client_id`/`client_secret` on its Integrations page, and each may need API access enabled separately. A missing Canada credential is a per-instance enablement gap, not a code bug.
- Full troubleshooting catalog: `references/gotchas.md`. Onboarding/data-migration mechanics: `references/implementation-lessons.md`. **Fastest first integration: `references/quickstart.md`** (ordered token → customer → invoice → PDF → test payment → webhook path, with a runnable Postman collection in `assets/`).

## Reference files (load on demand)

- `references/endpoints.md` — full endpoint catalog (method + path + purpose) and the incremental-sync query pattern.
- `references/webhooks.md` — webhook setup, event structure, retry logic, event types.
- `references/sandbox-testing.md` — sandbox base URL + test cards, ACH, bank-login, MFA credentials.
- `references/gotchas.md` — real-world troubleshooting (scrubbed, generalized).
- `references/implementation-lessons.md` — project/architecture lessons from real ERP integrations (credential handoff, SSO scoping, timelines, auto-generated-JE required fields).
- `references/quickstart.md` — the ordered first-integration path (token → customer → receivable → PDF → test payment → webhook), with a “Worked when” checkpoint per step, the four-IDs table, and the top reconciliation pitfalls. Official, partner-shareable.
- `assets/paystand-x-quickstart.postman_collection.json` — runnable version of the quickstart; drop in `client_id`/`client_secret`/`customer_id` from Dashboard → Integrations and it walks the whole flow including a test payment.
- `references/connector-pattern.md` — the canonical worked example: iPaaS/ERP connector ownership matrix, v1 data movement, and the **happy-path journal entries** (clearing-account model) for the full event→ERP-posting flow.

## What this API does NOT do today

See `references/endpoints.md` → “Not supported today.” Highlights: no merchant-wide bulk list of receivable transactions (per-receivable only), no date-range filter on fees or credit memos, no sub-day time windows (calendar-day only), no hard delete. The API does not apply credit memos itself: the payer applies an active credit memo at checkout (when credit memo checkout is enabled for the merchant), or you apply it in the ERP and re-sync the updated credit memo and receivable.

## Scope note

This is the **PSX API** skill (system-to-system REST). It is separate from CSV/dashboard workflows and from the `vulcan-psx` column-mapping skill. Keep customer names, ticket numbers, and internal URLs OUT of this skill — it is written to be shareable with external developers.
