# Lenco Sandbox Test Data

Use `LENCO_ENVIRONMENT=sandbox` and the sandbox base URL for all of these.

## Mobile money test numbers (verified against Lenco's docs, 2026-08-06)

| Phone | Operator | Result | Error |
|---|---|---|---|
| 0961111111 | mtn | Successful | - |
| 0962222222 | mtn | Failed | Not enough funds |
| 0963333333 | mtn | Failed | Withdrawal amount limit exceeded |
| 0964444444 | mtn | Failed | Transaction unauthorized |
| 0965555555 | mtn | Failed | Transaction unauthorized |
| 0966666666 | mtn | Failed | Transaction timed out |
| 0971111111 | airtel (zm) | Successful | - |
| 0972222222 | airtel (zm) | Failed | Incorrect PIN |
| 0973333333 | airtel (zm) | Failed | Invalid amount |
| 0974444444 | airtel (zm) | Failed | Payment invalid |
| 0975555555 | airtel (zm) | Failed | Not enough funds |
| 0976666666 | airtel (zm) | Failed | Failed |
| 0977777777 | airtel (zm) | Failed | Transaction timed out |
| 0978888888 | airtel (zm) | Failed | Failed |
| 0881111111 | tnm | Successful | - |
| 0883333333 | tnm | Failed | Not enough funds |
| 0885555555 | tnm | Failed | Transaction unauthorized |
| 0991111111 | airtel (mw) | Successful | - |
| 0992222222 | airtel (mw) | Failed | Not enough funds |
| 0984444444 | airtel (mw) | Failed | Transaction unauthorized |

Worth testing more than the happy path + one failure: the airtel (zm) set alone covers 6 distinct failure modes (PIN, amount validation, payment-invalid, funds, generic-failed, timeout) — if your error mapping only branches on Lenco's `errorCode` (01-13) rather than these operator-level messages, run a couple of these to confirm nothing gets swallowed into a generic "failed" bucket your user can't act on. Timeout in particular is worth checking against your reconciliation job, not just your webhook handler — a timed-out transaction is exactly the case where relying on the webhook alone can leave you stuck.

## Test cards

| Type | Number | CVV | Expiry |
|---|---|---|---|
| Visa | 4622 9431 2701 3705 | 838 | Any future date |
| Visa | 4622 9431 2701 3747 | 370 | Any future date |
| Mastercard | 5555 5555 5555 4444 | Any 3 digits | Any future date |

## What to actually test

- Happy path: successful collection → webhook fires → status updates correctly
- Failure path: failed collection (insufficient funds number above) → your error mapping surfaces the right `errorCode`
- Duplicate reference: submit the same reference twice → confirm you get error code `04`, not a silent double-processed transaction
- Webhook signature: send a request to your webhook endpoint with a *wrong* signature → confirm the guard rejects it (401/403), doesn't 500, doesn't process the payload
- Reconciliation: manually mark a transaction `pending` in your DB and delay/skip the webhook → confirm your polling job picks it up and resolves it
- `pay-offline` flow: mobile money collections normally return this, not a final status — the customer authorizes on their handset afterwards. Make sure your UI/state machine handles the intermediate state rather than assuming the POST response is final.
- Unknown status handling: feed your parser a status string it doesn't recognize (e.g. `otp-required`, which older Lenco docs listed) and confirm it doesn't crash or get silently mapped to "failed". Unknown should mean "still pending, let reconciliation decide."

## Webhook tests worth writing

These are the ones that catch real bugs, and they're cheap because you can POST to your own endpoint:

- **Wrong signature** → guard rejects with 401/403, does not 500, does not process the payload
- **Malformed/short signature header** (`X-Lenco-Signature: x`) → clean rejection, not a `RangeError` 500 from `timingSafeEqual`
- **Missing signature header entirely** → rejected
- **Valid signature, replayed twice** → the second one is deduped; balances/wallets change exactly once
- **Raw-body integrity**: send a payload containing a non-ASCII character (e.g. a name with an accent) with a signature computed over the raw bytes → confirm verification still passes. This is the case where a `JSON.stringify(req.body)` implementation silently breaks.
- **Slow handler**: confirm the endpoint returns 200 in well under a second even when the queued work is slow — Lenco treats a timeout as unacknowledged and retries for 24h
