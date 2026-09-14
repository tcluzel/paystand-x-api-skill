# PSX API — Quick Start for Integration Builders

> Distilled from Paystand's official “Paystand X API: Quick Start” (Product-owned, partner-shareable). The ordered path for a **first** integration. Do ALL of this in **sandbox** (`https://api.paystand.co/v3`) — a sandbox token is invalid in production and vice-versa. A runnable version is in `assets/paystand-x-quickstart.postman_collection.json`.

**Goal:** one webhook, in your handler, for a payment you made against an invoice you created. That single loop exercises the whole integration in miniature: auth → write → payer action → event back. Budget ~30 minutes.

**You need:** `client_id`, `client_secret`, `customer_id` (Dashboard → Integrations; sandbox & prod are separate sets) · a public HTTPS webhook endpoint with a valid cert (webhook.site / ngrok are fine for the first run; plain HTTP and self-signed certs are rejected) · a sandbox dashboard login (to send the invoice and pay it).

**Every authenticated call = two headers:** `Authorization: Bearer <token>` **and** `X-CUSTOMER-ID: <customer_id>`. Omitting `X-CUSTOMER-ID` returns **401 — identical to a bad token**. First call 401s with a fresh token? This is almost always why.

## Step 1 — Token
`POST /oauth/token` body `{grant_type: client_credentials, client_id, client_secret, scope: "auth"}`.
**Worked when:** 200 with `access_token`, `token_type: Bearer`, `expires_in: 1209600` (14 days). Refresh on a schedule (~every 13 days), not per request.

## Step 2 — Create a customer (must exist before any receivable)
`POST /payerCustomers` body `{extCustomerId, customerName, email, contactFirstName, contactLastName}`.
`extCustomerId` = your ERP primary key (≤40 chars, unique per merchant); Paystand **deduplicates on it** and returns it on every webhook. Use the real ERP key, NOT the returned `id` (that's Paystand's internal UUID).
**Worked when:** 201 and the response contains an `id`.

## Step 3 — Create a receivable
`POST /receivables/create` body `{extCustomerId, erpId, erpRef, totalAmount, amountDue, currency, postingDate, dueDate, status}`.
- `erpId` = internal invoice key, never shown to payer, **immutable after create** (wrong → new receivable is the only fix). `erpRef` = human-readable invoice number the payer sees.
- Link the customer with **`extCustomerId` OR `payerCustomerId`, never both.** `amountDue` ≤ `totalAmount`. Currency `USD` or `CAD`.
- **Fields come back renamed** (see gotchas.md): `erpId→extId`, `totalAmount→amount`, `amountDue→` consumed as `amountPaid = totalAmount−amountDue`, `postingDate→date`, `dueDate→dateDue`.
**Worked when:** 201 with `amount: 100.00`, `amountPaid: 0`, `status: active`.

## Step 4 — Attach the invoice PDF
`POST /receivables/<receivableId>/attachments` multipart `attachment=@invoice.pdf`. **One file per call, PDF only** (extension + `%PDF-` magic bytes validated), **up to 3 per receivable, 20 MB each.**
**Worked when:** 201 with `object: receivableAttachment`; `GET /v3/receivables/<id>` then lists it under `attachments`. (Use `GET /v3/receivables/<id>` — same shape as create; `…/read` returns a different, older field set.)

## Step 5 — Register webhook, then pay
Dashboard → Integrations → Webhook Event URLs. Every registered URL gets **every** event (no per-URL filtering) — one handler, route on `resource.object`. Then from the merchant dashboard send the receivable to the test payer and pay with card `4242424242424242` (any future expiry, any CVC).

## Step 6 — Confirm the events (four families, order NOT guaranteed)
| Event | Tells you |
|---|---|
| **Payment** | `created → processing → posted → paid`. `resource.amount` = gross the payer was charged. |
| **Receivable Transaction** | Payment applied to a specific invoice. `resource.receivable.extId` = your invoice; `amountApplied` = what landed on it. |
| **Fee** | Paystand's **merchant** processing cost. Immediate `paystand` event, or a `delayed` one later finalized by the real amount. |
| **Transfer** | Payout to the bank: `processing → sending → posted → paid`. |

**Worked when:** your handler logged a Payment reaching `paid` AND a Receivable Transaction whose `receivable.extId` is your invoice. Loop closed.
Cross-check: `GET /v3/receivables/<id>/transactions` (standard list envelope `{results, count, settings}`, `limit`/`offset`/`order`). **Waiting on a Transfer event?** Pay with card `4000000000000077` — funds skip the holding period and the payout fires the same day.

## The four IDs (Paystand's IDs → paths; yours → bodies)
| ID | Whose | Where |
|---|---|---|
| `extCustomerId` | Yours | Request bodies (customer + receivable create). Same name both ways. |
| `payerCustomer.id` | Paystand UUID | Path params: `/payerCustomers/:id` |
| `erpId` | Yours | Receivable create body only. Immutable; returned as `extId`. |
| `extId` | Yours, renamed | Every receivable response & webhook — **match on this**. |
| `receivable.id` | Paystand | Path params: attachments, transactions |
| `event.id` | Paystand | **Your idempotency key** |
| `paymentId` | Paystand | Groups the per-invoice events of one checkout |

**One-liner:** you write `erpId`, you read `extId` — same value.

## What NOT to build on day one
Skip polling (subscribe to events). Skip credit memos, refunds, disputes until the happy path closes. Skip transfer reconciliation until you've seen a payment land. Don't chase a webhook bug before confirming your endpoint is reachable from the public internet with a valid cert — that's the cause more often than your code. Hand each event to a queue and return 2xx within 15 s (inline ERP writes are why handlers time out).
