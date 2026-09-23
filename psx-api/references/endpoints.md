# PSX API — Endpoint Catalog

Base URL: `https://api.paystand.com/v3` (prod) / `https://api.paystand.co/v3` (sandbox).
All authenticated calls require `Authorization: Bearer <token>` + `X-CUSTOMER-ID: <merchant_customer_id>` + `Accept: application/json`.
Full interactive docs (with request/response schemas and code samples in ~20 languages): https://paystand-developers.netlify.app/docs/paystand-x-api

## Authentication
| Method | Path | Purpose |
|---|---|---|
| POST | `/oauth/token` | Generate access token (client_credentials, scope `auth`). Valid 2 weeks. |

## Merchant
| Method | Path | Purpose |
|---|---|---|
| GET | `/customers/timezone` | Get merchant default timezone (IANA). May 400 with Integrations OAuth on some merchants → supply a known IANA tz instead. |

## Customers (payers)
| Method | Path | Purpose |
|---|---|---|
| POST | `/payerCustomers` | Create payer customer. Returns `payerCustomer.id` and a nested `payer.id` (funding-source id — used on bank/payment calls; NOT the same as `payerCustomerId`). |
| GET | `/payerCustomers/:payerCustomerId` | Get one payer customer. |
| PUT | `/payerCustomers/:payerCustomerId` | Update payer customer. |
| GET | `/payerCustomers` | List payer customers. |

## Receivables (invoices)
| Method | Path | Purpose |
|---|---|---|
| POST | `/receivables/create` | Create receivable. Link with `payerCustomerId` OR `extCustomerId` (one required; customer must exist first). `erpId` and `erpRef` are REQUIRED. Fields are renamed on response (see gotchas.md). |
| GET | `/receivables/:receivableId/read` | Get one receivable — **same shape create returned** (renamed fields). Prefer this to assert against what you created. |
| GET | `/receivables/:receivableId` | Get one receivable — **older/different field set** than create returned (not in the public reference). |
| PUT | `/receivables/:receivableId/update` | Update a receivable. |
| POST | `/receivables/:receivableId/cancel` | Cancel (void) → `status: cancelled`. Record remains. |
| GET | `/receivables` | List receivables (filtered; Pattern A date filters). |
| GET | `/receivables/read` | Bare list — only `limit`/`offset`/`order` (no date filters). |
| GET | `/receivables/:receivableId/transactions` | Per-receivable transaction history (payment/refund applications). No merchant-wide bulk equivalent. |

## Payer banks & direct payments (merchant-initiated / tokenized model)
| Method | Path | Purpose |
|---|---|---|
| POST | `/payers/:payerId/banks` | Add a bank (ACH) to a payer. Creates `dropped:false, verified:false`. **Put your ref in `meta.externalId`, NOT top-level `externalId`** (top-level triggers a Stripe StripeSetupIntent failure — see gotchas.md). |
| POST | `/banks/:bankId/drops` | Send the two micro-deposits for verification (arrive ~1–2 business days). |
| PUT | `/banks/:bankId/drops` | Confirm micro-deposit amounts as whole-cent strings, e.g. `["32","45"]`. Order-independent. |
| POST | `/payments/secure` | Submit a payment against a verified bank (`bankId`) OR saved card (`cardId`), plus `payerId`, `amount`, `currency`. Optional `receivableId` applies it to an invoice. To surcharge the payer, also pass `feeSplit: {subtotal, feeSplitType}` from a prior `splitFees` call, taken from the rail that matches the payment method (`cardPayments` for a card, `networkBankPayments` for a verified bank; see gotchas.md) — without it, no convenience fee is applied. |

This is the backbone of the **merchant-triggered payment** model (merchant keeps the ledger, tells Paystand when/how much to charge): create payer → add & verify bank (or tokenize a card) → charge on demand via the payment API → use webhooks for success/failure → use transfers/withdrawals to track funds. Distinct from Paystand **Autopay** (which auto-charges on the receivable due date). Any saved payment method (bank or card) can also be used for Autopay once the payer approves it. For payer-driven “push” payments, send a **payment link** (see main skill) instead.

## Attachments (PDF on receivables)
| Method | Path | Purpose |
|---|---|---|
| POST | `/receivables/:receivableId/attachments` | Upload a PDF (multipart/form-data, field `attachment`). **One file per call, 20 MB each.** When the merchant account enforces PDF-only attachments, the file content must be a PDF (the extension is not checked). The 3-attachment limit is enforced on files sent with Create Receivable, not here. Returns `receivableAttachment` id. |
| DELETE | `/receivables/:receivableId/attachments/:attachmentId` | Remove a PDF from the invoice (does not delete the receivable). |

**Bulk/programmatic PDF attach limits** (from real high-volume integrations): no per-file count limit, but each **batch ≤ 100 MB** (individual files or a single ZIP); process **one batch at a time per customer** (start the next after the current completes); ~400 invoices per 100 MB ZIP at ~250 KB/file; ~300 PDFs/day is feasible within these limits. **Credit-memo PDFs are not supported.**

## Credit memos
| Method | Path | Purpose |
|---|---|---|
| POST | `/creditMemos` | Create a credit memo. Requires `extKey`, `status`, numeric `amount`/`amountRemaining`, MM-DD-YYYY dates. |
| GET | `/creditMemos` | List credit memos (filter with `f.querytype=own`, `by-payerCustomerId`, or `by-erpName` — NO date-range filter). |
| GET | `/creditMemos/:creditMemoId` | Get one credit memo. |
| PUT | `/creditMemos/:creditMemoId` | Update a credit memo. |
| PATCH | `/creditMemos/:creditMemoId/cancel` | Cancel a credit memo. |
| PATCH | `/creditMemos/:creditMemoId/activate` | Activate a credit memo. |

The API does NOT apply credit memos to receivables itself. Two paths: the payer applies an active credit memo at checkout (when credit memo checkout is enabled for the merchant), or you apply it in the ERP and re-sync the updated credit memo and receivable.

## Fees
| Method | Path | Purpose |
|---|---|---|
| GET | `/fees` | List merchant processing fees. May 401 under Integrations OAuth → fall back to embedded `fees[]` on Get Payment, or Fee Events. |
| POST | `/feeSplits/splitFees` | Split payment principal vs payer-side fees. `subtotal` + `currency` quotes the merchant's default plan; add `provider: receivable`, `payerId`, and `invoiceId` (receivable ID) to resolve the plan for a specific receivable. |
| POST | `/feeSplits/computeFees` | Estimate payer fee rates from Fees & Incentives plan (pre-payment). |

Merchant processing fees (cost to merchant) ≠ payer fees (recoup/surcharge added at checkout). Map `payerTotalFees` vs `merchantTotalFees` vs `fees[].amount` carefully in the ERP.

## Payments / refunds / disputes
| Method | Path | Purpose |
|---|---|---|
| GET | `/payments/:paymentId` | Get one payment (read `feeSplit`, embedded `fees[]`). |
| GET | `/payments/all` | List payments (Pattern A date filter; also Pattern B `startDate`/`endDate`). `posted` is NOT fee-final. |
| GET | `/refunds` / `/refunds/all` | List refunds (filtered + unfiltered, Pattern A). |
| GET | `/refunds/:refundId` | Get one refund. |
| GET | `/disputes` / `/disputes/all` | List disputes (filtered + unfiltered, Pattern A). |
| GET | `/disputes/:disputeId` | Get one dispute. |

## Transfers (payouts)
| Method | Path | Purpose |
|---|---|---|
| GET | `/transfers` | List transfers (filtered, Pattern A). Deposit/payout reconciliation. |
| GET | `/transfers/all` | List transfers (unfiltered). |
| GET | `/transfers/:transferId` | Get one transfer. |
| GET | `/transfers/:transferId/report` | Get the transfer report — wait until `status: completed`. |
| GET | `/transfers/:transferId/report/entries` | **The individual payments + fees inside a payout** — map each row into the ERP bank feed. Requires `settings.transfers.report.enabled`. This is how you reconcile a lump deposit to its line items. |

---

## Incremental sync — date-range query pattern (polling)

Use when polling instead of webhooks (on-prem ERPs without inbound HTTPS).

**Pattern A — nested `f.*` filters.** Date filtering ONLY applies when you set `f.querytype=by-query-filters`. Without it, other `f.*` params are ignored.

| Param | Required | Description |
|---|---|---|
| `f.querytype` | Yes | Must be `by-query-filters` |
| `f.datepreset` | for custom ranges | `custom` (with start/end) or a preset like `past7days` |
| `f.datestart` | for custom | `YYYY-MM-DD` |
| `f.dateend` | for custom | `YYYY-MM-DD` (inclusive calendar day) |
| `f.datetype` | recommended | `created` or `lastUpdated` — use `lastUpdated` for incremental sync |
| `f.limit` | no | page size (default 50) |
| `f.offset` | no | pagination |
| `f.order` | no | e.g. `lastUpdated DESC` |
| `f.timezone` | no | IANA name; affects calendar-day → UTC conversion. Get from Get Merchant Timezone. |

Example:
```
GET /v3/receivables?f.querytype=by-query-filters&f.datepreset=custom&f.datestart=2026-05-01&f.dateend=2026-05-22&f.datetype=lastUpdated&f.limit=50&f.offset=0
```

Pattern A supported on: `/receivables`, `/payments/all`, `/refunds` (`/all`), `/disputes` (`/all`), `/transfers`, `/withdrawals` (`/all`).

**Pattern B — top-level `startDate`/`endDate`** on some list endpoints (e.g. `/payments/all`).

**Only calendar-day granularity** — no sub-day (hour) windows.

---

## Not supported today (public API)

| Capability | Status | Use instead |
|---|---|---|
| Merchant-wide bulk list of receivable transactions | Not available | Per-receivable `/receivables/:id/transactions` or Receivable-Transaction webhooks |
| `view=export` on fees / receivable transactions | Not partner-ready | List Fees, fee webhooks, nested receivable transaction routes |
| Date-range filter on fees | Not supported | Fee Events, or paginate `/fees` and filter client-side by payment ID |
| Date-range filter on credit memos | Not supported | `/creditMemos` with `f.querytype=own` / `by-payerCustomerId` / `by-erpName` |
| Sub-day time windows | Not supported | Calendar-day `f.datestart`/`f.dateend` only |
| Hard DELETE receivables / payer customers | Not supported | Cancel receivable; update status to `cancelled` |
| Facilitator/platform tokens | Not required | Plain merchant API credentials |
| **Create/save a CARD** (`POST …/cards`) | **Not in the public API** | Obtain a `cardId` via the hosted **Save Payment Method** embed / tokenization link (PCI reasons), then charge with `POST /payments/secure` using `cardId`. Only **Add Bank Account** (`POST /payers/:id/banks`) is documented for saving a funding source. |
| Auto-surcharge the convenience fee on an API charge | Not automatic (only at hosted checkout) | Call `POST /feeSplits/splitFees`, then charge the `payerTotal` of the rail that matches the payment method via `POST /payments/secure` |
