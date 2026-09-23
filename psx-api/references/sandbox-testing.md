# PSX API — Sandbox & Test Credentials

These are Paystand's PUBLIC sandbox test values (published on the developer docs). Sandbox transactions are simulated — no real funds.

## Sandbox environment

- **Base URL:** `https://api.paystand.co/v3`  (`.co`, not `.com`)
- Auth: Integrations OAuth (`POST /oauth/token`, `scope: auth`) + your merchant `X-CUSTOMER-ID` from Dashboard → Integrations.
- Public docs are validated against Paystand's dummy sandbox merchant.
- **Known Integrations-token gaps** (other routes work on same merchant): `GET /fees`, `GET /fees/{id}`, `POST /fees/compute`, `GET /feeSettingPlan/{id}`, `GET /customers/timezone`. Use documented fallbacks (embedded `fees[]` on Get Payment, etc.).
- Pre-payment payer fees: `subtotal` + `currency` quotes the default plan; call Split Fees with `provider: receivable`, `payerId`, `invoiceId` (receivable ID) to resolve the plan for a specific receivable.

## Sandbox payment limits (simulated)

| Method | Limit |
|---|---|
| Credit Card | $5,000 |
| ACH | $2,000 |
| Bank Network | $1,200 |

## Test credit cards

| Card Number | Result |
|---|---|
| `4000000000000077` | Succeeds; funds added directly to available balance |
| `4242424242424242` | Charge succeeds |
| `4000000000000002` | Declined — card declined |
| `4000000000000069` | Declined — expired card |
| `4000000000000127` | Declined — incorrect CVC |
| `4000000000000259` | Succeeds, then later disputed as fraudulent |

Expiration `12/28`, CVV `123` for generic card tests.

## Test bank accounts (ACH)

| Account Number | Routing Number | Result |
|---|---|---|
| `000123456789` | `110000000` | Transitions posted → paid (USD) |
| `000123456789` | `11000-000` | Transitions posted → paid (CAD) |

## Test bank-network (bank login) credentials

All use username `testuser` / password `testpass`.

| Bank | BankKey | MFA Question | MFA Answer |
|---|---|---|---|
| Wells Fargo | `wellsfargo` | You say tomato, I say...? | `tomato` |
| Bank of America | `bankofamerica` | What is the name on the bank account? | `paystand` |
| Chase | `chase` | What is the account holder type? | `individual` |
| All Banks | `allbanks` | Account number | `9876543210` |

## MFA answers reference

| MFA Question | Answer |
|---|---|
| You say tomato, I say...? | `tomato` (returns another question-type MFA to test chaining) |
| What is the name on the bank account? | `paystand` |
| What is the account holder type? | `individual` or `company` |
| Account number | `9876543210` |
| Routing number | `111121111` |

Source: https://paystand-developers.netlify.app/docs/paystand-x-api/other/testing-credentials
