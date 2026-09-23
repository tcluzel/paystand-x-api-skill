# PSX API — iPaaS / ERP Connector Reference Pattern

> A reusable reference pattern for a merchant moving onto an ERP with **no native Paystand integration**, connected via an iPaaS/middleware platform. The ownership split and journal-entry model apply to any PSX-API + iPaaS + ERP build. This is the canonical worked example of the full **event → ERP posting** flow.

## When this pattern applies

Merchant's ERP has no native Paystand connector, so an iPaaS/middleware sits between the ERP and the PSX API. **The ERP remains the book of record.** Customers + posted invoices sync ERP → Paystand so payers can pay; payment/fee/settlement events post back Paystand → ERP. v1 scope is typically **invoice-based AR only** (customers, invoices, PDFs); product/item and sales-order sync are out of scope unless asked.

## Ownership matrix (agree this at kickoff)

| Responsibility | Paystand | iPaaS/Connector | Merchant |
|---|---|---|---|
| Dashboard, OAuth credentials, webhook payloads | **Owns** | — | — |
| Connector logic, retries, idempotency, ERP writes | — | **Owns** | — |
| Chart of accounts, ERP access, posting policy | Advises | Advises | **Owns** |
| Payer experience, fees, collections in Paystand | **Owns** | — | Configures |

Sandbox and production are separate credential sets (`client_id`, `client_secret`, `customer_id` from Dashboard → Integrations). Build in sandbox until UAT.

## v1 data movement

- **ERP → Paystand:** create the customer before its receivables. Customers key on `extCustomerId`, receivables on `erpId` (required; changeable later only by a deliberate update, and webhooks match on it) — pick the right keys. Sync invoices on post. Up to 3 PDFs per receivable. Currency USD or CAD.
- **Paystand → ERP:** subscribe to webhooks (don't poll); use read endpoints to backfill after an outage. Delivery is at-least-once → **dedupe on event `id`.**

| Event | Use it for | Key fields |
|---|---|---|
| Receivable Transaction | Applying cash to a specific invoice | `receivable.extId` (your invoice), `amountApplied` (invoice portion only) |
| Payment | Gross collected + payer fee split | `amount` (gross), `feeSplit.subtotal`, `feeSplit.payerTotalFees` |
| Fee | Merchant processing cost | `feeType` — post only when `paystand`; a `delayed` event has no reliable amount |
| Transfer | Deposit reconciliation | `amount`, `dateSettled` |
| Refund | Reversal (per merchant policy) | `paymentId`, `feesRefunded` |

## Happy-path journal entries (the reconciliation model)

Example: **$100 invoice, $3 convenience fee, $2.50 merchant fee, $100.50 deposit.**

| Event | Debit | Credit |
|---|---|---|
| Invoice posted in ERP | AR $100 | Income $100 |
| Payment (invoice + convenience fee) | Paystand clearing $103 | AR $100 **and** CF income $3 |
| Merchant fee | Merchant-fee expense $2.50 | Paystand clearing $2.50 |
| Daily settlement | Bank/settlement $100.50 | Paystand clearing $100.50 |

**The clearing account nets to zero for the payment.** Key facts baked into this model:
- Copying an invoice into Paystand does **not** create an ERP journal — the ERP already posted it.
- The **convenience fee is derived from the payment**, not a Fee event: `amount` ($103) is gross, `feeSplit.subtotal` ($100) applies against the invoice, `feeSplit.payerTotalFees` ($3) posts to a fee-income account.
- Payment, fee, and settlement arrive as **separate events at different times**, so a receivable can be **closed in the ERP while cash is still in the clearing account** — that's expected, not a bug.
- **The merchant fee is a separate, later posting.** It is not known at payment time and arrives a short while after (in practice ~30 min), via Fee data / `GET /fees` / embedded `fees[]` — NOT the payment webhook. Post it as its **own JE to a dedicated processing-fee GL account** when it finalizes; don't fold it into the payment receipt at capture. The daily transfer already excludes it, so this JE is what closes the clearing account.
- **One GL account per fee type** — convenience fee, merchant processing fee, discounts/incentives, disputes — so the connector can split each cleanly. The accounting team owns the small gain/loss between the convenience fee collected and the merchant fee charged.
- Settlement is **once a day, one lump** covering every payment that day (not once per payment).
- If a payer settles several invoices in one checkout, you get one Receivable Transaction per invoice sharing a `paymentId` — **sum `amountApplied`; do not also post the parent `payment.amount`** (double-count).

## Kickoff checklist

1. ERP sandbox company/tenant + an iPaaS user with access to the objects above.
2. Paystand X sandbox credentials + public HTTPS webhook URL.
3. Implement the loop: customer → receivable → PDF → test payment → cash application → transfer.
4. Scope decisions to make explicitly: credit memos, AR Advance (unapplied cash), discounts/incentives, dispute write-back — all supported by the platform; decide which are in v1 and what the ERP posting should be. Name the GL accounts (AR, CF income, merchant-fee expense, clearing/settlement); the connector maps events to them.
