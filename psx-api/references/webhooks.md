# PSX API — Webhooks

Webhooks are HTTP POST callbacks Paystand sends when AR events occur (payment applied, deposit reconciled, etc.). Use them instead of polling when your stack can receive inbound HTTPS.

## Setup

**Prerequisites:** a publicly accessible HTTPS endpoint with a valid SSL cert, able to accept POST + JSON and respond `2xx` within **15 seconds**.

1. Build an endpoint that accepts POST, processes `application/json`, and returns HTTP 200–299.
2. In the **Dashboard → Integrations → Webhook Event URLs**, enable events and add one or more URLs.
3. Test with webhook.site (quick) or ngrok (local dev).

## Event structure

```json
{
  "object": "event",
  "id": "unique_event_id",
  "resource": {
    "object": "payment",   // payment | receivableTransactions | refund | dispute | transfer | fee | creditMemo
    "id": "...",
    "status": "paid"
    // ...resource data
  },
  "diff": { "previous": { }, "changes": { } },
  "urls": ["your_webhook_url"],
  "sent": false,
  "attempts": 0,
  "sourceId": "resource_id",
  "sourceType": "ResourceType",
  "status": "active",
  "created": "2025-07-11T21:34:56.000Z",
  "lastUpdated": "2025-07-11T21:34:57.000Z"
}
```

The `diff` block tells you *what changed* (previous state + changes) — use it for idempotent ERP updates.

## Response requirements

- Return HTTP **200–299** for success. Anything outside that range = failure → triggers retry.
- Respond within **15 seconds** (request timeout).

## Auto-retry

**8 attempts total**, fixed intervals:

| Retry | Delay |
|---|---|
| 1st | 300s (5 min) |
| 2nd | 900s (15 min) |
| 3rd | 3600s (1 hr) |
| 4th | 43200s (12 hr) |
| 5th–8th | 86400s (24 hr each) |

**Stops retrying immediately** on `404 Not Found` or invalid-URL errors. Design your endpoint to always return 2xx once received (process async) so a slow downstream ERP doesn't burn retries.

## Event types (webhook page + REST polling fallback)

| Webhook resource | REST list fallback | Date filter? | Notes |
|---|---|---|---|
| Payment | `GET /payments/all` | Yes (A + B) | `posted` is not fee-final |
| Fee (merchant processing) | `GET /fees` | No | May 401 under Integrations OAuth → use embedded `fees[]` on Get Payment. Merchant fees only, not payer recoup. |
| Receivable transaction | `GET /receivables/:id/transactions` | No (bulk) | Per-receivable only; no merchant-wide bulk |
| Refund | `GET /refunds` (`/all`) | Yes | |
| Dispute | `GET /disputes` (`/all`) | Yes | |
| Transfer | `GET /transfers` (filtered) | Yes | Payout reconciliation |
| Credit memo | Credit Memo APIs | — | |
| Withdrawal | `GET /withdrawals` (filtered) | Yes | No dedicated public webhook page |

Prefer webhooks where possible; fall back to the polling matrix (see `references/endpoints.md`) when inbound HTTPS isn't available.
