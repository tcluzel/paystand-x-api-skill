# PSX API — Real-World Gotchas & Troubleshooting

Generalized from production integration issues (customer names, ticket numbers, and internal URLs removed — safe to share externally).

## Auth & scope

- **`insufficient_scope` / 403 has TWO common causes** (verified by reproducing a partner's exact failing calls). A request missing EITHER returns `insufficient_scope`:
  1. **Token requested without `scope: "auth"`.** Omitting `scope` returns a token that *looks* valid but has no scope attached, and every subsequent call 403s. A correct token response **echoes `"scope": "auth"`** back — if that field is absent, the token is dead. Always send `scope:"auth"` in the `/oauth/token` body.
  2. **Missing `X-CUSTOMER-ID` header.** The Bearer token alone is NOT sufficient — `X-CUSTOMER-ID` must accompany it on every call.
  Also confirm API keys are enabled on the merchant's API Plan Settings (`show_api_keys`) in Dashboard → Integrations.
- **401 on Get Customer** → you're using the wrong customer ID. `payerCustomer.id` is NOT the same as `payment.payerId` unless that payer is listed under your merchant. Sending an ID that isn't under your merchant returns 401.
- **401 on `GET /fees` / `GET /feeSettingPlan/{id}`** under Integrations OAuth is a known gap on some merchants. Fall back to embedded `fees[]` on Get Payment or Fee Event webhooks.

## Adding a payer bank (ACH) — the `externalId` → StripeSetupIntent trap + full flow

- **Do NOT send `externalId` at the TOP LEVEL of the bank object.** On `POST /payers/:payerId/banks`, a top-level `externalId` is interpreted as a **Stripe payment-method ID**, which triggers a verification flow and fails bank creation before the account is saved — you get `"An error occurred while creating StripeSetupIntent"` and an HTTP 400 `apiError` with the **same `ref` on every retry** (a repeating ref = this exact bug, not a transient error).
- **Fix:** move your reference into **`meta.externalId`** instead. `meta` is stored and returned when you retrieve the bank, so you can use it as your external reference **without maintaining a separate mapping**. The rest of a normal payload is fine (`country:"USA"`, `accountHolderType:"company"|"individual"`, `billingAddress`, `accountType`, `routingNumber`, `accountNumber`, `currency`).
- **Full bank-funded payment flow** (micro-deposit verification):
  1. `POST /payers/:payerId/banks` → creates the bank with `dropped:false, verified:false`.
  2. `POST /banks/:bankId/drops` → sends two micro-deposits (typically arrive 1–2 business days).
  3. `PUT /banks/:bankId/drops` → confirm the amounts as **whole-cent strings**, e.g. `["32","45"]` for $0.32 / $0.45. **Order does not matter.**
  4. `POST /payments/secure` → submit the payment using `bankId`, `payerId`, `amount`, `currency`.

## ACH failure is two-stage (design your failure handling around this)

- **Stage 1 — synchronous:** invalid/wrongly-formatted bank details are rejected immediately by the API at save/charge time; the account isn't saved and you get the error inline.
- **Stage 2 — asynchronous:** if details are valid, the ACH goes to the bank and the payment shows `posted`; it can still be **returned/charged back days later** (e.g. NSF). Paystand notifies the **merchant** by email + dashboard with the reason — **NOT the payer.** To alert your customer and send a manual pay link, orchestrate it yourself (detect via notification/API, then send a Paystand payment link).

## Receivable create — field names change between request and response (READ THIS)

The single most common “the response is missing the field I just sent” confusion: **every receivable field you send comes back under a different name.** Nothing is missing — it is renamed (or consumed) on write.

| You send | Comes back as |
|---|---|
| `erpId` | `extId` |
| `totalAmount` | `amount` |
| `amountDue` | **consumed, not returned** — stored as `amountPaid = totalAmount − amountDue` |
| `postingDate` | `date` |
| `dueDate` | `dateDue` |
| `erpRef` | not returned on create (stored internally as `invoiceKey`) |

Consequences:
- **Match incoming webhooks/responses on `extId`** (= the `erpId` you sent). “You write `erpId` and you read `extId` — they are the same value.”
- **Compute the open balance as `amount − amountPaid`.** There is no `amountDue` on any response. A fully-unpaid invoice comes back `amountPaid: 0` (not `amountDue: 100`). Creating one with `amountDue: 0` comes back `paid` (surprising during backfills).
- **`erpId` is IMMUTABLE after create.** Get it wrong and the only remedy is a new receivable.
- Link the receivable to a customer with **EITHER `payerCustomerId` OR `extCustomerId`, never both**; the customer must exist first. `amountDue` must be ≤ `totalAmount`. Currency is `USD` or `CAD`.
- (`invoiceId` is the older name for this key and is OPTIONAL; current docs use `erpId`. If a mapping layer still enforces `invoiceId` as required, that's stale validation.)

## `GET /receivables/:id` vs `/read` return different shapes

- `GET /v3/receivables/:id` returns the **same shape create returned** (the renamed fields above) — use this to assert against what you created.
- `GET /v3/receivables/:id/read` returns a **different, older field set.** Don't mix them up when writing tests.

## ID chaining (store the right ID from each response)

| ID | Where you get it | Use for |
|---|---|---|
| Merchant `customer_id` | Dashboard → Integrations | `X-CUSTOMER-ID` header on EVERY call |
| `payerCustomer.id` | Create/List/Get Customer | `payerCustomerId` on receivables; credit-memo scope. ≠ `payment.payerId`. |
| `extCustomerId` | Your ERP customer key | Alternative to `payerCustomerId` on receivable create/update |
| `receivable.id` | Create/Get/List Receivable | Attachments, cancel, transactions |
| `payment.id` | Payment webhook / List Payments | Fee lookup (`fees[]`), payer fee split (`feeSplit`), refunds, reconciliation |
| `fee.id` | Fee webhook / List Fees | GL lines for merchant processing fees (not payer recoup) |
| `transfer.id` | List/Get Transfer | Payout reconciliation |
| `creditMemo.id` | Credit memo APIs | Cancel/activate credit memos |

## Reconciliation nuances

- **The gross and the applied amount are different numbers.** On a $100 invoice with a $3 convenience fee, `payment.amount` = `103.00` but `receivableTransaction.amountApplied` = `100.00`. Both are correct: apply `amountApplied` against the invoice; derive the payer fee from `payment.feeSplit`.
- **The convenience fee is NOT a Fee event.** Fee events carry the **merchant's** processing cost only. The **payer-facing convenience fee lives on `payment.feeSplit`**: `feeSplit.payerTotalFees` = the fee, `feeSplit.subtotal` = the invoice portion. There is no top-level convenience-fee field. Two different fees, two accounts, two sources.
- **A `delayed` fee has NO usable amount — don't post it.** Card processing fees depend on card tier and are finalized after the banking partner confirms. Record the amount **only when `feeType` is `paystand`**, and overwrite if a later `paystand` event arrives for the same fee `id`. A `delayed` fee event is a placeholder.
- **One checkout → many Receivable Transaction events.** If a payer settles three invoices at once you get three events sharing one `paymentId`. **Sum the `amountApplied` values.** Summing those *and* the parent `payment.amount` **double-counts the money** — the single most common reconciliation bug on this API.
- **Payment `posted` status is not fee-final.** Don't post final GL fee lines off a `posted` payment.
- **Credit memos are not applied by the API.** Apply them in the ERP, then re-sync the receivable with the updated amount. Negative receivable amounts are not supported. (Credit-memo endpoints DO exist for lifecycle ops — see endpoints.md.)

## Webhook delivery & idempotency

- **Delivery is at-least-once — you WILL get duplicates.** That's the design. **Deduplicate on the event `id`** before applying anything. Use the event `id` as your idempotency key.
- **Timestamps are authoritative; arrival order is NOT.** Don't build logic assuming Payment lands before its Receivable Transaction. A card payment produces **four event families** (Payment, Receivable Transaction, Fee, Transfer), not necessarily in order.
- **Return 2xx within 15 seconds.** Hand the event to a queue and return immediately; doing the ERP write inline is why most handlers time out under load. Retries: 5 min, 15 min, 1 h, 12 h, then 24 h ×4 — 8 attempts over ~4 days. A **404 from your endpoint stops retries immediately.**
- Registered webhook URLs receive **every** event (no per-URL filtering) — use one handler and route internally on `resource.object`.

## Rate limits & retry-safety

- **3,000 requests/minute per API key**, with limit headers on every response. Retry `5xx` and `429` with exponential backoff + jitter.
- **Writes deduplicate on your external IDs**, so every write is safe to retry. “If a retry would worry you, you have a bug.”

## Access token lifecycle

- Tokens are valid **1,209,600 s = 14 days**. **Refresh on a schedule (~every 13 days), NOT per request.** A naive per-request integration works for two weeks and then fails silently. The `refresh_token` field is reserved for future use — just re-request a token.

## Checkout URL behavior (context, not core API)

- Checkout/statement links can accept either the Paystand customer ID or the ERP customer ID (when the option is enabled) for the **statement view**. The **receivable view** may not honor the ERP ID the same way — test both before assuming symmetry.

## Timezone / calendar days

- `GET /customers/timezone` can 400 under Integrations OAuth on some merchants. If so, pass a known IANA timezone (from merchant settings) via `f.timezone` on filtered lists instead of relying on the lookup.
- Date filters are **calendar-day only** — no hour-level windows. `f.dateend` is inclusive.
