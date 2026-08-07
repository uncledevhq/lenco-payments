# Lenco Collections — Types & Patterns

Read this when taking money **in** (customer payments). Endpoint paths and filters are in `api-reference.md`; this is the TypeScript layer, settlement semantics, and the card/3DS flow.

Verified against Lenco's OpenAPI spec, 2026-08-06.

## The collection object

Returned by all five collection endpoints (list returns an array of it):

```typescript
interface LencoCollection {
  id: string;
  initiatedAt: string;                  // ISO8601 UTC
  completedAt: string | null;           // null until resolved
  amount: string;                       // decimal string, e.g. "10.00"
  fee: string | null;                   // null until the collection completes
  bearer: 'merchant' | 'customer';      // who pays the fee
  currency: string;                     // 'ZMW'
  reference: string | null;             // yours
  lencoReference: string;               // Lenco's
  type: 'card' | 'mobile-money' | 'bank-account' | null;
  status: 'pending' | 'successful' | 'failed' | 'pay-offline' | '3ds-auth-required';
  source: 'banking-app' | 'api';
  reasonForFailure: string | null;
  settlementStatus: 'pending' | 'settled' | null;
  settlement: LencoSettlement | null;
  mobileMoneyDetails: {
    country: string;
    phone: string;
    operator: string;
    accountName: string | null;
    operatorTransactionId: string | null;   // the operator's own txn ref — keep it, see below
  } | null;
  bankAccountDetails: null;             // always null in current spec
  cardDetails: {
    firstName: string | null;
    lastName: string | null;
    bin: string | null;                 // first 6 — safe to store
    last4: string | null;               // safe to store
    cardType: string | null;            // e.g. 'Mastercard'
  } | null;
  meta?: {                              // card only, when status is 3ds-auth-required
    authorization: { mode: 'redirect'; redirect: string };
  };
}

interface LencoSettlement {
  id: string;
  amountSettled: string;                // amount MINUS fee, when bearer is merchant
  currency: string;
  createdAt: string;
  settledAt: string | null;
  status: 'pending' | 'settled';
  type: 'instant' | 'next-day';
  accountId: string;                    // which of your accounts got credited
}
```

`amount`, `fee`, and `amountSettled` are decimal **strings** — same money-handling rules as transfers, see `transfers.md`.

## Two statuses that mean different things

`status` and `settlementStatus` are independent and both matter:

- `status: "successful"` = the customer's money left their account. Deliver the goods on this.
- `settlementStatus: "settled"` = the money reached *your* Lenco account. Reconcile your books on this.

A collection can be `successful` with `settlementStatus: "pending"` and `settlement: null` for a while. If your ledger writes only on `successful`, your Lenco balance and your books will disagree during that window — usually fine, but decide deliberately which event drives which side of your accounting rather than discovering it during a month-end reconciliation.

`settlement.type` is `instant` or `next-day`. Next-day settlement means a real gap between "customer paid" and "we have the money," which matters for cash-flow modelling if you're settling out to vendors on the back of incoming collections.

## Fees and the `bearer` field

`bearer` decides who absorbs the fee:
- `merchant` (default) — you eat it. Customer pays `amount`, you receive `amount - fee`. In the docs' example: amount `10.00`, fee `0.25`, `amountSettled` `9.75`.
- `customer` — the customer is charged the fee on top.

`fee` is `null` on the initial POST response (the collection hasn't completed, so it isn't computed yet) and populated once resolved. Don't display a fee from the POST response; it won't be there.

For card collections, the docs note `bearer` in the payload is **only used if not already set in your dashboard** — dashboard config wins. Worth checking what yours is set to rather than assuming the API value applies.

### The fee asymmetry (collections vs transfers)

Fees sit on opposite sides of the two directions:

- **Collections** (money in): fee is **deducted** from what reaches you — customer pays `10.00`, you settle `9.75` (with `bearer: merchant`).
- **Transfers** (money out): fee is **added** to what leaves you — you send `20.00`, your account is debited `20.00 + fee`.

So a full collect-and-payout cycle loses margin on *both* legs. If you're building anything that passes money through (marketplace, payroll on top of collections, agent float), model both fees or your unit economics silently overstate.

### Fee rates are data, not constants

Neither the API nor this skill carries Lenco's fee *schedule* — rates live on Lenco's pricing page and dashboard, and can be account-specific. Engineering consequences:

- **Never hardcode a fee percentage** in code or tests. Read the actual `fee` off each completed collection/transfer and reconcile from actuals.
- If you must show an estimated fee *before* initiating (checkout UX, payout previews), source the rate from your own config, label it an estimate, and true it up from the real `fee` after completion — remember `fee` is `null` until then.
- Watch `fee` in reconciliation: if Lenco changes your rate, the per-transaction `fee` field is where you find out. A drift alert ("fee ≠ expected rate ± tolerance") turns a silent margin change into a ticket.

## Mobile money collections

```typescript
// POST /collections/mobile-money
// required: amount, reference, phone, operator
{
  amount: number;                                       // major units, JSON number
  reference: string;                                    // unique; - . _ + alphanumeric
  phone: string;
  operator: 'airtel' | 'mtn' | 'tnm' | 'zamtel';        // zm: airtel|mtn|zamtel  mw: airtel|tnm
  country?: 'zm' | 'mw';
  bearer?: 'merchant' | 'customer';                     // default merchant
}
```

Note there is **no `accountId`** — unlike transfers, you don't nominate which account receives the money in the request. Settlement destination comes back in `settlement.accountId`.

The POST typically returns `status: "pay-offline"`, meaning the customer must now authorize on their handset. `completedAt`, `fee`, `settlementStatus`, and `settlement` are all null at this point. Resolve the real outcome via webhook or by polling `/collections/status/:reference`.

```typescript
export class InitiateMobileMoneyCollectionDto {
  @IsNumber({ maxDecimalPlaces: 2 }) @IsPositive() amount: number;

  @IsString() @Matches(/^[a-zA-Z0-9._-]+$/) @MaxLength(100) reference: string;

  @IsString() phone: string;

  @IsIn(['airtel', 'mtn', 'tnm', 'zamtel']) operator: 'airtel' | 'mtn' | 'tnm' | 'zamtel';

  @IsOptional() @IsIn(['zm', 'mw']) country?: 'zm' | 'mw';

  @IsOptional() @IsIn(['merchant', 'customer']) bearer?: 'merchant' | 'customer';
}
```

**Keep `mobileMoneyDetails.operatorTransactionId`.** It's the mobile money operator's own reference (e.g. `MP240312.0000.A00001`). When a customer says "I paid, MTN sent me a confirmation SMS" and your system shows nothing, that ID is what support uses to trace it with the operator. Store it on your payment record when it appears; it's not recoverable later if you drop it.

## Card collections

Requires PCI DSS certification. The request body is a single field:

```json
{ "encryptedPayload": "<JWE string>" }
```

The plaintext you encrypt (RSA-OAEP-256 + A256GCM, key from `GET /encryption-key`, don't cache it):

| Field | Required | Notes |
|---|---|---|
| `email` | yes | customer email |
| `reference` | yes | unique, **case sensitive**, `-` `.` `_` + alphanumeric |
| `amount` | yes | may include decimals (10.75) |
| `currency` | yes | ISO 3-letter, e.g. `ZMW`, `USD` |
| `bearer` | no | `merchant` \| `customer`; dashboard setting takes precedence |
| `customer.firstName` / `customer.lastName` | yes | |
| `billing.streetAddress` / `.city` / `.postalCode` / `.country` | yes | `country` is 2-letter (`US`) |
| `billing.state` | no | 2-letter for US states / CA provinces; blank where N/A |
| `card.number` / `.expiryMonth` / `.expiryYear` / `.cvv` | yes | never log any of these |
| `redirectUrl` | no | see 3DS below |

Note the card `reference` is documented as **case sensitive**, which isn't stated for the other endpoints. Normalize your reference generation to one case so you can't create `Ref-1` and `ref-1` as distinct collections by accident.

### 3DS flow

If the card needs 3D Secure, the response comes back with `status: "3ds-auth-required"` and a `meta.authorization` object:

```json
"meta": { "authorization": { "mode": "redirect", "redirect": "https://pay.lenco.co/auth/..." } }
```

Redirect the customer to that URL. After they complete it, Lenco sends them to your `redirectUrl` with `reference`, `lencoReference`, `status`, and optional `errorMessage` appended as query parameters.

**Treat those query params as untrusted.** They arrive via the customer's browser, so anyone can hand-craft a URL with `status=successful`. Use them only to decide what to show on screen; always re-verify server-side with `GET /collections/status/:reference` before releasing goods or crediting an account. Same rule as the widget's `onSuccess` callback.

## List filters

```typescript
export class ListCollectionsQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number = 1;
  @IsOptional() @IsDateString() from?: string;   // YYYY-MM-DD
  @IsOptional() @IsDateString() to?: string;
  @IsOptional() @IsIn(['pending', 'successful', 'failed', 'pay-offline']) status?: string;
  @IsOptional() @IsIn(['card', 'mobile-money', 'bank-account']) type?: string;
  @IsOptional() @IsString() country?: string;
}
```

`3ds-auth-required` is absent from the filterable status enum even though collections can be in that state — so you can't page for abandoned 3DS attempts directly. Track those in your own DB from the POST response if you need to report on drop-off.

For reconciliation, `status=pay-offline` plus a `from` date gives you the mobile money collections still waiting on customer authorization.

## Known doc bugs (Lenco's side, not yours)

- The `/collections/card` response example shows `"type": "mobile-money"` — should be `"card"`. The prose schema above it is correct.
- The card payload sample shows `"amount": "1000"` as a string while the mobile money endpoint types `amount` as a JSON number. The card body is encrypted so the spec can't validate it; if a string amount is rejected in sandbox, try a number.
