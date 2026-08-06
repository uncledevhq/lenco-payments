# Lenco API Reference

## Base URLs
- Production: `https://api.lenco.co/access/v2`
- Sandbox: `https://api.sandbox.lenco.co/access/v2`
- Payment widget (prod): `https://pay.lenco.co/js/v1/inline.js`
- Payment widget (sandbox): `https://pay.sandbox.lenco.co/js/v1/inline.js`

## Auth
`Authorization: Bearer YOUR_API_TOKEN` on every request. HTTPS only. Get/rotate tokens via support@lenco.co.

## Response envelope
Every response follows this shape — **always check `status` (boolean), not just the HTTP code**. Lenco can return HTTP 200 with `status: false` for a business-logic failure (e.g. insufficient funds on a transfer that was still "successfully submitted" at the HTTP layer).

```typescript
{
  status: boolean;       // true if successful — this is the real success signal
  message: string;       // log it, don't branch on it (not stable across versions)
  data: object | array;
  meta?: { total: number; perPage: number; currentPage: number; pageCount: number };
}
```

## HTTP status codes
| Code | Meaning |
|---|---|
| 200/201 | Success |
| 400 | Validation/client error |
| 401 | Unauthorized (bad/missing token) |
| 404 | Not found |
| 500-504 | Server error — report immediately, don't silently retry-loop |

## Error codes (branch on these, from `data.errorCode`, not on `message` text)
| Code | Meaning |
|---|---|
| 01 | Validation error |
| 02 | Insufficient funds |
| 03 | Transfer limit exceeded |
| 04 | Invalid/duplicate reference |
| 05 | Invalid recipient account |
| 06 | Restriction on debit account |
| 07 | Invalid/duplicate bulk transfer reference |
| 08 | Invalid number of objects in bulk transfer |
| 09 | Invalid auth token / authorization denied |
| 10 | General error |
| 11 | Resource not found |
| 12 | Invalid mobile number |
| 13 | Access to resource denied |

## Endpoints

### Accounts
- `GET /accounts` — supports `?page=N` (default 1), paginated response includes `meta` (see Response envelope). Response `data` is an array of accounts.
- `GET /accounts/:id` — `:id` is your 36-char account UUID. Response `data` is a single account object (same shape as above, unwrapped from the array).
- `GET /accounts/:id/balance` — response `data`: `{ availableBalance: string, ledgerBalance: string, currency: string }`

**Verified gotcha:** on `/accounts/:id` and `/accounts/:id/balance`, a 400 ("Account was not found or api key does not have access to the account") returns `data: []` (empty array), not `data: null` like the resolve endpoints do. Don't write a single generic "if !status, data is null" error handler across all Lenco calls — the shape of `data` on failure isn't consistent across endpoints. Check `status` and branch on the error, but don't assume anything about what `data` contains when `status` is false beyond "don't trust it."

### Banks & resolution
- `GET /banks?country=zm`
- `POST /resolve/bank-account`
- `POST /resolve/mobile-money`
- `POST /resolve/lenco-money`
- `POST /resolve/lenco-merchant`

### Transfer recipients
- `GET /transfer-recipients` — query params: `page` (default 1), `type` (`mobile-money` | `bank-account` | `lenco-money` | `lenco-merchant`), `country` (e.g. `ng`, `zm`). Returns array + `meta`.
- `GET /transfer-recipients/:id` — `:id` is a 36-char recipient UUID. 400 → `data: null`, message "Transfer Recipient was not found"
- `POST /transfer-recipients/bank-account`
- `POST /transfer-recipients/mobile-money`
- `POST /transfer-recipients/lenco-money`
- `POST /transfer-recipients/lenco-merchant`

#### Create requests (verified 2026-08-06)
You never send `accountName` — Lenco resolves the name from the identifiers and returns it. A 400 here means "Account Details could not be verified", i.e. the identifiers didn't resolve to a real account. Same identifier fields as the matching `/resolve/*` endpoint:

```json
// bank-account   (required: accountNumber, bankId; optional: country)
{ "accountNumber": "9130000000000", "bankId": "002", "country": "zm" }

// mobile-money   (required: phone, operator; optional: country — only `zm`)
{ "phone": "0750000000", "operator": "zamtel" }   // operator: airtel | mtn | zamtel

// lenco-money    (required: walletNumber)
{ "walletNumber": "0000001" }

// lenco-merchant (required: tillNumber)
{ "tillNumber": "0000001" }
```

**Practical consequence:** creating a recipient already performs the resolve. If your flow is "confirm the name with the user, then save the recipient", you don't strictly need a separate `/resolve/*` call first — but you do if you want to show the name *before* persisting anything on Lenco's side. Pick one deliberately rather than doing both by accident on every save.

#### Recipient response shape
All four creates and both reads return the same envelope. Note the doubled `type` — once at the top level and once inside `details` — and that `details` is a discriminated union keyed on `details.type`:

```json
{
  "status": true,
  "message": "",
  "data": {
    "id": "d6b6e00e-bdb6-43a6-a561-85b61496198e",
    "currency": "ZMW",
    "type": "mobile-money",
    "country": "zm",
    "details": {
      "type": "mobile-money",
      "accountName": "Beata Jean",
      "phone": "0750000000",
      "operator": "zamtel"
    }
  }
}
```

`details` fields by type: `bank-account` → `accountNumber` + `bank: { id, name, country }`; `mobile-money` → `phone` + `operator`; `lenco-money` → `walletNumber`; `lenco-merchant` → `tillNumber`. On the **list** endpoint the docs type all of these as present-but-nullable on every record; on the single-recipient and create responses only the fields for that type come back. Discriminate on `details.type` rather than null-checking individual fields — see types.md.

### Transfers (sending money)
- `GET /transfers` — filters: `page`, `from` / `to` (YYYY-MM-DD), `search`, `accountId`, `transferRecipientId`, `type` (`mobile-money` | `bank-account` | `lenco-money` | `lenco-merchant`), `status` (`pending` | `successful` | `failed`), `country`. Returns array + `meta`.
- `GET /transfers/:id` — 36-char transfer UUID. **404** (not 400) with `data: null`, "Transfer was not found"
- `GET /transfers/status/:reference` — lookup by *your* reference. Also **404** on miss.
- `POST /transfers/bank-account`
- `POST /transfers/mobile-money` — Zambia and Malawi only
- `POST /transfers/lenco-money`
- `POST /transfers/lenco-merchant`
- `POST /transfers/account` — between your own Lenco accounts

#### Transfer request bodies (verified 2026-08-06)

All four outbound transfers require `accountId` (36-char UUID of the account to **debit**), `amount` (number, major units), and `reference` (unique, `-` `.` `_` + alphanumeric only). `narration` is optional.

The destination can be given **two ways** — either a saved `transferRecipientId`, or the raw identifiers inline. You don't need to pre-create a recipient:

```json
// bank-account   → transferRecipientId, OR accountNumber + bankId (+ optional country)
{ "accountId": "<uuid>", "amount": 20.00, "reference": "ref-3",
  "accountNumber": "9130000000000", "bankId": "002", "country": "zm" }

// mobile-money   → transferRecipientId, OR phone + operator (+ optional country)
// operator enum: airtel | mtn | tnm | zamtel   country: zm | mw
{ "accountId": "<uuid>", "amount": 20.00, "reference": "ref-4",
  "phone": "0750000000", "operator": "airtel", "country": "zm" }

// lenco-money    → transferRecipientId, OR walletNumber
{ "accountId": "<uuid>", "amount": 20.00, "reference": "ref-5", "walletNumber": "0000001" }

// lenco-merchant → transferRecipientId, OR tillNumber
{ "accountId": "<uuid>", "amount": 20.00, "reference": "ref-6", "tillNumber": "0000001" }

// account (own-to-own) → creditAccountId is REQUIRED, no recipient option
{ "accountId": "<debit-uuid>", "amount": 20.00, "reference": "ref-7",
  "creditAccountId": "<credit-uuid>" }
```

**Note the country asymmetry:** mobile money *transfers* support `zm` and `mw` (operator enum includes `tnm`), but `/resolve/mobile-money` and `/transfer-recipients/mobile-money` are `zm`-only with no `tnm`. So you can transfer to a Malawi number inline, but you can't pre-resolve or save it as a recipient. If you're building a Malawi flow, that's a real constraint, not an oversight on your part.

#### Transfer response

Same object from all eight endpoints (list returns an array of it):

```json
{
  "id": "9525b4c6-502b-45be-90e1-81eb81a3f424",
  "amount": "20.00",
  "fee": "8.50",
  "currency": "ZMW",
  "narration": "Transfer",
  "initiatedAt": "2024-01-01T00:00:00.447Z",
  "completedAt": "2024-01-01T00:00:01.237Z",
  "accountId": "b176cda5-7d97-4a3f-b4dd-ab0234e9e08c",
  "creditAccount": { "type": "bank-account", "accountName": "Beata Jean", "accountNumber": "...", "bank": { "id": "002", "name": "Absa Bank", "country": "zm" } },
  "status": "successful",
  "reasonForFailure": null,
  "reference": "ref-3",
  "lencoReference": "240010002",
  "extraData": { "nipSessionId": null },
  "source": "api"
}
```

- `amount` and `fee` are **strings**, and `fee` is charged **on top of** `amount` (example: 20.00 sent, 8.50 fee). Parse with a decimal library, never `parseFloat` for anything you'll sum or reconcile — see types.md.
- `creditAccount` is a discriminated union on `type`, same variants as recipient `details`, plus an `id` field (populated for `/transfers/account`, null elsewhere).
- `extraData.nipSessionId` is Nigeria-specific (NIP = Nigeria Inter-Bank Settlement System). Expect `null` on all Zambian traffic.
- `status` here is only `pending` | `successful` | `failed` — no `pay-offline`/`otp-required`, those are collection-side only.

#### "Duplicate reference" — read this before writing retry logic

A `POST` with an already-used `reference` returns **400 with `message: "Duplicate reference"`** (errorCode `04`) and `data: null`. It does **not** return the original transfer.

This is good — it's server-side protection against double-sending money. But it changes how you retry. If your HTTP call times out or the connection drops, you do **not** know whether Lenco processed it. Retrying blindly gets you one of two things:

- `400 Duplicate reference` → the first attempt **did** land. The money may have moved. You must now `GET /transfers/status/:reference` to find out what actually happened.
- A normal success → the first attempt never landed.

So the correct recovery path is: **on any ambiguous failure (timeout, network error, 5xx), don't retry the POST — query `/transfers/status/:reference` first.** Only re-POST if that returns 404 (transfer genuinely doesn't exist). Treating "Duplicate reference" as a plain error and surfacing it to a user as "transfer failed" is the dangerous bug here: the transfer may well have succeeded.

### Collections (receiving money)
- `GET /collections` — filters: `page`, `from` / `to` (YYYY-MM-DD), `status` (`pending` | `successful` | `failed` | `pay-offline` — note `3ds-auth-required` is **not** filterable), `type` (`card` | `mobile-money` | `bank-account`), `country`. Returns array + `meta`.
- `GET /collections/:id` — 36-char collection UUID. **404** with `data: null`, "Payment details was not found"
- `GET /collections/status/:reference` — lookup by your reference. Also **404** on miss.
- `POST /collections/mobile-money`
- `POST /collections/card` — PCI DSS required; body is a single JWE `encryptedPayload` string

Full request/response shapes, settlement semantics, and the card 3DS flow: `references/collections.md`.

### Settlements & transactions
- `GET /settlements` — filters: `page`, `from` / `to` (YYYY-MM-DD), `status` (`pending` | `settled`), `type` (`instant` | `next-day`), `collectionType` (`card` | `mobile-money` | `bank-account`), `country`
- `GET /settlements/:id` — 36-char settlement UUID. **404**, "Settlement was not found"
- `GET /transactions` — filters: `page`, `type` (`credit` | `debit`), `from` / `to`, `search`, `accountId`
- `GET /transactions/:id` — **404**, "Transaction was not found"

#### Settlement object
Same fields as the nested `settlement` on a collection, **plus an embedded `collection`**:

```json
{
  "id": "c04583d7-...", "amountSettled": "9.75", "currency": "ZMW",
  "createdAt": "2024-03-12T07:14:10.439Z", "settledAt": "2024-03-12T07:14:10.496Z",
  "status": "settled", "type": "instant",
  "accountId": "68f11209-...",
  "collection": { /* full collection object, see collections.md */ }
}
```

Two subtleties in that embedded collection:
- Its own `settlement` field is always `null` (Lenco breaks the circular reference). Don't read `settlement.collection.settlement`.
- Its `mobileMoneyDetails` **omits `operatorTransactionId`**, which the collections endpoints do include. If you need the operator's reference, fetch the collection directly rather than reading it off a settlement.

This means `GET /settlements` gives you settlement + collection in one call — for a payouts/reconciliation report that's one round trip instead of N+1 (settlements, then a collection lookup per row). Use it.

#### Transaction object
Plain ledger entries against your Lenco accounts — this is your statement, not a payment resource:

```json
{
  "id": "d6730fe6-...",
  "amount": "13.00",
  "currency": "ZMW",
  "narration": "Transfer / 240730006",
  "type": "debit",
  "datetime": "2024-01-10T14:24:31.931Z",
  "accountId": "b176cda5-...",
  "balance": "997559.00"
}
```

- `balance` is the **running balance after this entry** — useful for statement rendering, and it's how you'd detect a gap if your own ledger drifts.
- `narration` embeds the `lencoReference` (`"Transfer / 240730006"`). That's the only link back to the originating transfer/collection — there's no `transferId`/`collectionId` field. Parsing it is fragile; prefer matching on your own records keyed by `lencoReference` captured at initiation time.

### Encryption
- `GET /encryption-key` — returns the RSA public key as a **JWK**, not PEM:

```json
{ "status": true, "message": "", "data": { "kty": "RSA", "use": "enc", "n": "...", "e": "AQAB", "kid": "2bbb0d...2f68aa" } }
```

Feed it to a JOSE library directly (`jose`'s `importJWK`) rather than converting to PEM. Encrypt with **RSA-OAEP-256 + A256GCM** and include the `kid` in the JWE header so Lenco knows which key you used. Fetch it per request — don't cache; a cached key breaks silently when Lenco rotates.

## Resolve endpoints — verified request/response (2026-08-06)

All four return `data: null` with `message: "Account details was not found"` on a 400 if the account can't be resolved — check `status` before reading `data`, same as everywhere else.

### `POST /resolve/mobile-money`
Request:
```json
{ "phone": "0961111111", "operator": "mtn", "country": "zm" }
```
- `phone`, `operator` required. `operator` enum: `airtel`, `mtn`, `zamtel` (Zambia only — this endpoint currently only supports `zm`, unlike the wider Zambia/Malawi split elsewhere in the API)
- `country` optional, only `zm` currently supported

Response (200):
```json
{
  "status": true,
  "message": "",
  "data": {
    "type": "mobile-money",
    "accountName": "Beata Jean",
    "phone": "0750000000",
    "operator": "zamtel",
    "country": "zm"
  }
}
```

### `POST /resolve/bank-account`
Request:
```json
{ "accountNumber": "9130000000000", "bankId": "002", "country": "zm" }
```
- `accountNumber`, **`bankId`** required (not `bankCode` — get the `id` from `GET /banks?country=zm`, don't assume it's the same as a bank's public sort/branch code)
- `country` optional (e.g. `ng`, `zm`)

Response (200):
```json
{
  "status": true,
  "message": "",
  "data": {
    "type": "bank-account",
    "accountName": "Beata Jean",
    "accountNumber": "9130000000000",
    "bank": { "id": "002", "name": "Absa Bank", "country": "zm" }
  }
}
```

### `POST /resolve/lenco-money`
Request:
```json
{ "walletNumber": "0000001" }
```
Response (200):
```json
{
  "status": true,
  "message": "",
  "data": { "type": "lenco-money", "accountName": "Beata Jean", "walletNumber": "0000001" }
}
```

### `POST /resolve/lenco-merchant`
Request:
```json
{ "tillNumber": "0000001" }
```
Response (200):
```json
{
  "status": true,
  "message": "",
  "data": { "type": "lenco-merchant", "accountName": "Account Name", "tillNumber": "0000001" }
}
```

## Payment collection widget (frontend)
```javascript
<script src="https://pay.lenco.co/js/v1/inline.js"></script>

LencoPay.getPaid({
  key: 'YOUR_PUBLIC_KEY',       // public key, never the secret token
  reference: orderReference,     // generated by YOUR BACKEND from the order/payment ID — not a
                                 // timestamp or client-side random; see SKILL.md's idempotency note
  email: 'customer@email.com',
  amount: 1000,                  // major units — 10.75 not 1075
  currency: "ZMW",
  channels: ["card", "mobile-money"],
  customer: { firstName: "John", lastName: "Doe", phone: "0971111111" },
  onSuccess: (response) => { /* signal only — verify on backend via reference */ },
  onClose: () => {},
  onConfirmationPending: () => {},
});
```

## Payment methods
- **Zambia mobile money:** `airtel` (Airtel Money), `mtn` (MTN MoMo), `zamtel` (Zamtel Kwacha)
- **Malawi mobile money:** `airtel` (Airtel Money), `tnm` (TNM Mpamba)

## Status values
- Transfer: `pending`, `successful`, `failed`
- Collection: `pending`, `successful`, `failed`, plus `pay-offline` (mobile money) and `3ds-auth-required` (card)
- Account types: `bank-account`, `mobile-money`, `lenco-money`, `lenco-merchant`

### `otp-required` — NOT in the current spec (corrected 2026-08-06)
Older versions of Lenco's mobile money collection docs described an `otp-required` status, where Lenco texts the customer a code that you collect and submit before the payment proceeds (sandbox OTP `000000`). Third-party community SDKs still include it in their status enums.

**The current v2.0 spec does not list it** — `/collections/mobile-money` documents only `pending | successful | failed | pay-offline`, and the list/get endpoints add only `3ds-auth-required`. There is no Submit OTP endpoint in the sidebar.

Treat it as removed or deprecated. Practical guidance: don't build an OTP submission flow, but do make your status handling tolerant of an unrecognized status string rather than crashing or silently mapping it to "failed" — if Lenco re-enables it for some operator, a hard enum parse would break in production. Log unknowns and treat them as still-pending until reconciliation resolves them.

## Currency
Primary: **ZMW**.

## Reference format rules
Alphanumeric, `-`, `.`, `_` only. Must be unique per transaction — see SKILL.md's idempotency note for why this matters more than it looks.

## Common pitfalls (don't do these)
1. Storing API tokens in localStorage/sessionStorage on the frontend
2. Calling Lenco directly from the frontend instead of proxying through the backend
3. Reusing references across transactions
4. Converting amounts to lowest currency unit (kobo/cents) — Lenco wants major units
5. Relying solely on webhooks without a re-query/polling fallback
6. Committing API tokens to version control
7. Skipping webhook signature verification
8. Doing long-running work before responding to a webhook
