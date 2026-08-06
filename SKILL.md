---
name: lenco-payments
description: Integrate with the Lenco API for payments in Zambia (mobile money — Airtel/MTN/Zamtel — bank transfers, cards, collections, transfers, settlements). Use this skill whenever the user mentions Lenco, mobile money payments, ZMW payment collection, payment transfers/collections/settlements in a Zambian/Malawian fintech context, or asks to build/debug a payments integration in a NestJS backend — even if they don't say "Lenco" by name. Covers auth, endpoints, webhooks + signature verification, error codes, TypeScript types, and sandbox test data.
license: MIT
metadata:
  author: Chomba Chanda (uncledev)
  status: unofficial, community-maintained
  verified-against: Lenco API v2.0 docs, 2026-08-06
---

# Lenco Payments Integration

By Chomba Chanda (uncledev) · MIT · Unofficial, not affiliated with Lenco.

*Last checked against https://lenco-api.readme.io/v2.0/reference (v2.0) — 2026-08-06. Every schema here is transcribed from Lenco's published OpenAPI spec, but Lenco ships no changelog, so drift is silent. Treat that date as the confidence marker: recent means trust it, months old means spot-check the endpoint you're about to use against the live docs.*

Reference + implementation patterns for integrating Lenco (api.lenco.co) into a NestJS backend. Lenco handles mobile money (Airtel/MTN/Zamtel in Zambia, Airtel/TNM in Malawi), bank transfers, and card collections, all denominated in ZMW (or MWK).

## When to load what

- Writing/debugging any Lenco API call, DTO, or service method → read `references/api-reference.md` for the exact endpoint, request/response shape, and error codes. Don't guess endpoint paths from memory — this file is the source of truth.
- Sending money out (transfers, payouts, disbursements) → `references/transfers.md` for types, DTOs, money handling, and the safe retry/recovery pattern. Read this before writing any transfer code — the duplicate-reference semantics are not intuitive and getting them wrong can double-pay.
- Taking money in (customer payments, checkout, mobile money collections, card/3DS) → `references/collections.md` for types, settlement semantics, the `pay-offline` flow, and the card payload. Read this before writing collection code — `status` and `settlementStatus` mean different things and conflating them breaks reconciliation.
- Defining TypeScript interfaces/DTOs for accounts, collections, resolve, or recipients → `references/types.md`
- Building or debugging the webhook receiver → `references/webhooks.md`. Read it before writing the handler: Lenco's own Node and PHP examples disagree on what gets signed, the event envelope differs from the API envelope, and the obvious `timingSafeEqual` usage crashes on a malformed header.
- Writing sandbox tests → `references/testing.md` for known test numbers/cards and their expected outcomes

## Core architecture pattern (NestJS)

Never call Lenco directly from a frontend or from a controller — always proxy through a dedicated module. Standard shape:

```
src/lenco/
├── lenco.module.ts
├── lenco.service.ts          — wraps HttpService, all outbound calls live here
├── lenco.config.ts           — env-driven config (token, base URL, sandbox/prod)
├── dto/
│   ├── initiate-collection.dto.ts
│   ├── initiate-transfer.dto.ts
│   └── resolve-account.dto.ts
├── webhooks/
│   ├── lenco-webhook.controller.ts
│   └── lenco-webhook.guard.ts   — verifies x-lenco-signature before the handler runs
└── lenco.types.ts             — see references/types.md
```

**Why a dedicated module, not inline HTTP calls in whatever service needs payments:** Lenco is a hard external dependency with its own auth, error shape, and retry semantics. Isolating it means (a) one place to swap sandbox↔prod, (b) one place to mock in tests, (c) the rest of your app depends on your own clean interface, not Lenco's response envelope. This is the same reason you wouldn't scatter `prisma.$queryRaw` calls across every service — centralize the external boundary.

### Config

```typescript
// lenco.config.ts
export const lencoConfig = registerAs('lenco', () => ({
  apiToken: process.env.LENCO_API_TOKEN,
  publicKey: process.env.LENCO_PUBLIC_KEY,
  baseUrl: process.env.LENCO_ENVIRONMENT === 'production'
    ? 'https://api.lenco.co/access/v2'
    : 'https://api.sandbox.lenco.co/access/v2',
  webhookHashKey: crypto.createHash('sha256').update(process.env.LENCO_API_TOKEN!).digest('hex'),
}));
```

Validate these at startup (Zod or class-validator on a config class) — don't let the app boot with a missing `LENCO_API_TOKEN` and fail on the first payment attempt instead.

### Service pattern

Every outbound method should: build the request, call Lenco, check `response.status` (not just HTTP 200 — see api-reference.md, Lenco can 200 with `status: false`), map errors to your own domain exceptions, and never leak the raw Lenco response to callers.

```typescript
async initiateMobileMoneyCollection(dto: InitiateCollectionDto): Promise<Collection> {
  const { data } = await firstValueFrom(
    this.http.post('/collections/mobile-money', dto, { headers: this.authHeaders }),
  );
  if (!data.status) {
    throw new LencoApiException(data.message, data.errorCode);
  }
  return this.mapToCollection(data.data);
}
```

`LencoApiException` should carry the Lenco `errorCode` (see api-reference.md's error code table) so upstream code can branch on "insufficient funds" vs "invalid recipient" vs "general error" without string-matching messages.

**Don't assume a consistent `data` shape on failure.** Verified against the actual API: some endpoints return `data: null` on error (resolve endpoints), others return `data: []` (account lookups). Your error mapping should only ever read `message` and `errorCode` — never touch `data` when `status` is `false`.

### Idempotency (verified behavior — get this right before shipping transfers)

Every transfer/collection needs a caller-generated unique `reference`. Lenco rejects a reused one with a 400 `"Duplicate reference"` (errorCode `04`) and does **not** return the original transaction.

Two consequences:

1. **Derive the reference deterministically from your own domain entity** (payout ID, order ID) — not a timestamp, not a fresh UUID per attempt. If the reference changes between retries, Lenco's duplicate check can't protect you and you can double-pay.
2. **Never blind-retry a transfer POST after a timeout or 5xx.** You don't know whether it landed. Query `GET /transfers/status/:reference` first; only re-POST if that 404s. Treating `"Duplicate reference"` as "the transfer failed" and telling the user so is the dangerous version of this bug — the money may already be gone.

Full recovery pattern in `references/types.md`; endpoint details in `references/api-reference.md`.

### Mobile money collections are asynchronous

`POST /collections/mobile-money` typically returns `status: "pay-offline"`, not a final result. That means the customer still has to authorize on their handset. Notify them, then resolve the outcome via webhook or by polling `/collections/status/:reference`. Never treat the POST response as proof of payment.

(Older Lenco docs described an extra `otp-required` step; it's not in the current spec — see `references/api-reference.md` before building anything around it.)

### Verification flow (don't trust client callbacks)

1. Frontend widget calls `onSuccess` → this is a UI signal only, not proof of payment
2. Frontend calls your backend with the `reference`
3. Backend calls `GET /collections/status/:reference` (or `/transfers/status/:reference`) against Lenco directly
4. Backend updates your DB based on *that* response, not the frontend's claim

### Don't rely on webhooks alone

Webhooks can fail to deliver. Pair webhook handling with a polling job (re-query pending transactions on an interval — e.g. a BullMQ repeatable job) until status resolves to `successful` or `failed`. See `references/webhooks.md` for the full webhook + reconciliation pattern.

## Security checklist (OWASP-relevant, always check before shipping)

- [ ] `LENCO_API_TOKEN` only in env vars, never committed, never returned in any API response
- [ ] All Lenco calls go through the backend — public key only, never the secret token, reaches the frontend
- [ ] Webhook signature verified against the **raw** request body (not re-serialized JSON), in a guard, failing closed — see webhooks.md. This is the only auth on that route; without it anyone can POST a fake "payment successful"
- [ ] Webhook handler responds 200 fast and queues slow work — Lenco retries every 30 min for 24h on any non-2xx, so a slow handler causes duplicate events, not resilience
- [ ] Webhook worker is idempotent, deduped on `lencoReference` via a DB unique constraint (retries are guaranteed, and a double-processed `collection.successful` is a financial bug)
- [ ] References are unique per transaction and derived from your own entity IDs — see the idempotency section above
- [ ] Card collection payloads (if used) are JWE-encrypted per `/encryption-key` (JWK, RSA-OAEP-256 + A256GCM, fetched fresh not cached), and card details are never logged
- [ ] 3DS redirect query params (`status`, `reference`, ...) treated as untrusted — they arrive via the customer's browser; always re-verify server-side before releasing value
