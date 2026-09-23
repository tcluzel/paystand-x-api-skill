# PSX API — Real-World Gotchas & Troubleshooting

Generalized from production integration issues (customer names, ticket numbers, and internal URLs removed — safe to share externally).

## Auth & scope

- **`insufficient_scope` / 403 → the token was requested without `scope: "auth"`.** Omitting `scope` returns a token that *looks* valid but has no scope attached, and every subsequent call 403s with `insufficient_scope`. A correct token response **echoes `"scope": "auth"`** back — if that field is absent, the token is dead. Always send `scope:"auth"` in the `/oauth/token` body.
  Also confirm API keys are enabled on the merchant's API Plan Settings (`show_api_keys`) in Dashboard → Integrations.
- **Missing `X-CUSTOMER-ID` → 401 `insufficientResourceAccess`** (not 403). The Bearer token alone is NOT sufficient — `X-CUSTOMER-ID` must accompany it on every call.
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

## `POST /payments/secure` — unapplied-payment & Sage Intacct caveats

- **Omitting `receivableId` creates an UNAPPLIED payment.** It is a generic payment not linked to any receivable ledger entry, ERP-sync event, or AR-Advance entitlement — it is **not** the same as an AR Advance. If your plan tracks AR and you need advance/prepayment tied to the ledger, use AR Advance, not a bare `/payments/secure`.
- **Sage Intacct-connected merchants: `receivableId` is REQUIRED.** Without it the request returns a validation error — Sage merchants must route payments through a receivable (Create Receivable + linked payment) so cash applies correctly in Sage.

## Saving a payment method: BANK has a REST API, CARD does not

- **Bank/ACH — fully documented REST.** `POST /payers/:payerId/banks` → micro-deposit verify (`…/banks/:bankId/drops`) → charge via `POST /payments/secure` with `bankId`. (Flow above.)
- **Card — there is NO documented endpoint to create/save a card.** The public reference has **no `Add Card` / `POST …/cards` operation** — the Payers section documents only *Add Bank Account*. You **cannot POST raw card details as JSON.**
  - You obtain a `cardId` (a saved-card token) through the **hosted “Save Payment Method” embed or tokenization link** (payer enters the card on Paystand's page). This is deliberate — it keeps raw card PANs off your servers and out of your PCI-DSS scope.
  - Once you have the token, **charging a saved card IS a documented API**: `POST /payments/secure` with `cardId` + `payerId` + `amount` + `currency`.
  - If you need a server-to-server card-tokenization API specifically, that is not in the public docs — request it from your Paystand contact. Do not assume a `POST /payers/:id/cards` body exists.

## Merchant-initiated charges are NOT auto-surcharged — it's TWO calls: quote, then pay

At **hosted checkout**, Paystand resolves the Fees & Incentives plan and adds the payer convenience fee automatically. Over the **API** (`POST /payments/secure`), **the plan is NOT consulted** — Paystand charges **exactly the `amount` you send**. Verified: against a plan with a $10 flat card fee, a $100 charge sent with no `feeSplit` came back `payerTotalFees: 0.00`, `payerTotal: 100.00` — the fee never applied and the merchant absorbed the processing cost.

The convenience fee is a **two-call flow**: **quote → pay.** Ask `splitFees` what the payer owes, then send that back on the payment. Skip the first call and there's no fee; make it but don't pass the result along and there's *still* no fee.

**1. Quote — `POST /v3/feeSplits/splitFees`** (for a merchant on their default plan, `subtotal` + `currency` is the whole request):
```json
{ "subtotal": "100.00", "currency": "USD" }
```
Returns a per-rail breakdown, e.g. a plan set to 3.5% on card:
```json
{
  "cardPayments": {
    "feeType": "cardPayment.posted",
    "feeSplitType": "recoup_custom_of_subtotal",
    "customRate": "0.035", "customFlat": "0.00",
    "subtotal": "100.00", "payerTotalFees": "3.50", "payerTotal": "103.50"
  },
  "achBankPayments": { "feeSplitType": "absorb_all_fees", "payerTotalFees": "0.00", "payerTotal": "100.00" }
}
```
(For AR/receivable context, also pass `provider: receivable`, `payerId`, `invoiceId` so the right plan resolves.)

**2. Pay — `POST /v3/payments/secure`** — charge the `payerTotal` and echo the split back:
```json
{
  "cardId": "...", "payerId": "...",
  "amount": "103.50", "currency": "USD",
  "feeSplit": { "subtotal": "100.00", "feeSplitType": "recoup_custom_of_subtotal" }
}
```
- **The `feeSplit` you pass back carries `subtotal` + `feeSplitType` only — do NOT echo `customRate`/`customFlat`.** The server re-derives the rate from the plan, so there's less for the integration to get wrong.
- If the plan is **absorb** (`absorb_all_fees`), `splitFees` returns `payerTotalFees: 0` and the merchant eats the cost — so "call `splitFees` and charge the `payerTotal` it returns" is always correct, as long as you take it from the right rail (next point).
- **Use the rail that matches the payment method.** `splitFees` quotes every rail at once because what a plan charges differs by method. Charge the `payerTotal` and echo the `feeSplitType` of the rail you are actually charging: `cardPayments` for a card, `networkBankPayments` for a verified bank (`bankId`). A verified bank's rail carries discounts only, never a payer fee, so charging it with another rail's quote (e.g. `achBankPayments`) is rejected with a generic `400 apiError`.
- **Multiple plans:** to target a specific plan rather than the merchant's default, pass **`feesIncentivesUrlKey`** (or `feeSettingPlanId`) on **both** calls.

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
- **`erpId` can be changed after create** with `PUT /receivables/:id/update` (send the full body — a partial one fails on the missing `erpRef`). Webhooks match on it, so treat a change as a deliberate re-key, not a routine update.
- Link the receivable to a customer with **EITHER `payerCustomerId` OR `extCustomerId`, never both**; the customer must exist first. `amountDue` must be ≤ `totalAmount`. Currency is `USD` or `CAD`.
- **`erpId` and `erpRef` are required.** A missing one returns `400 parameterMissing` under its stored name (`extId` / `invoiceKey`). There is no `invoiceId` alias.

## `GET /receivables/:id` vs `/read` return different shapes

- `GET /v3/receivables/:id/read` returns the **same shape create returned** (the renamed fields above) — use this to assert against what you created.
- `GET /v3/receivables/:id` returns a **different, older field set** (`status: current`, `dateDue` as `MM-DD-YYYY`, no `date`). Don't mix them up when writing tests.

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
- **The merchant processing fee arrives LATER and separately — post it as its own journal entry.** The merchant fee is **not known at payment time**; it's finalized and retrievable a short while after the payment (in practice ~30 min later), and it comes via a **different call than the payment** (Fee data / `GET /fees` / embedded `fees[]`, not the payment webhook). Don't try to fold it into the payment receipt at capture. Recommended pattern (per Paystand Product): post the **full payment** and the **payer convenience fee** at payment time, then when the merchant fee finalizes, create a **separate JE to a dedicated processing-fee GL account**. The end-of-day **transfer/payout already nets out the merchant fee** (the deposit excludes it), so the ERP-side merchant-fee JE is what makes the clearing account reconcile.
- **One GL account per fee type.** Map each to its own account so the connector can split cleanly: **convenience fee**, **merchant processing fee**, **discounts/incentives**, **disputes**. The merchant's accounting owns the profit/loss delta between the convenience fee collected and the merchant fee charged (they may gain or lose a few cents per txn — that's expected, not a bug).
- **A break-even convenience fee must recoup BOTH the % and the flat.** Paystand's processing fee is `percentage + flat` (e.g. `3.5% + $0.35`). To break even, the merchant's convenience fee must match **both** components — a percentage-only surcharge (e.g. "3.5%") silently loses the per-transaction flat portion on every payment. When advising a merchant on their Fees & Incentives config, mirror the flat amount too.
- **Credit memos are not applied by the API itself.** Either the payer applies an active credit memo (`amountRemaining` > 0, same currency as the invoice) at checkout when credit memo checkout is enabled for the merchant, or you apply it in the ERP and re-sync: push the updated credit memo (`PUT /creditMemos/:id`, cancel/activate) and the receivable's new amount. Negative receivable amounts are not supported. (Credit-memo endpoints DO exist for lifecycle ops — see endpoints.md.)

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
