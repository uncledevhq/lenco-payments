# Lenco Transfers — Types & Patterns

Read this when writing or debugging outbound money movement (transfers, payouts). Endpoint paths, request bodies, and the duplicate-reference semantics are in `api-reference.md`; this file is the TypeScript layer and the safe-transfer pattern.


This supersedes the rough `LencoTransfer` sketch in `types.md` — use the version below.

```typescript
type LencoCreditAccount =
  | {
      type: 'bank-account';
      id: string | null;
      accountName: string;
      accountNumber: string;
      bank: { id: string; name: string; country: string };
    }
  | { type: 'mobile-money'; id: string | null; accountName: string; phone: string; operator: string; country: string }
  | { type: 'lenco-money'; id: string | null; accountName: string; walletNumber: string }
  | { type: 'lenco-merchant'; id: string | null; accountName: string; tillNumber: string };

interface LencoTransfer {
  id: string;
  amount: string;              // decimal string, e.g. "20.00" — NOT a number
  fee: string;                 // decimal string, charged ON TOP of amount
  currency: string;
  narration: string;
  initiatedAt: string;         // ISO8601 UTC
  completedAt: string | null;
  accountId: string;           // the debited account
  creditAccount: LencoCreditAccount;
  status: 'pending' | 'successful' | 'failed';
  reasonForFailure: string | null;
  reference: string | null;    // your reference
  lencoReference: string;      // Lenco's own reference
  extraData: { nipSessionId: string | null };  // Nigeria-only; null in ZM
  source: 'banking-app' | 'api';
}
```

### Money is a string — keep it that way

`amount` and `fee` arrive as decimal strings (`"20.00"`, `"8.50"`). Don't `parseFloat` them into your domain model. IEEE-754 doubles can't represent most decimal fractions exactly, so `0.1 + 0.2 !== 0.3`, and summing a few thousand transfer fees in floats gives you a reconciliation total that's off by cents — the kind of bug that surfaces as "our numbers don't match Lenco's statement" months later.

Options, in order of preference for your setup:
- **Store as integer minor units** (ngwee) in Postgres `BIGINT`, convert at the boundary. Fastest, exact, no dependency. `"20.00"` → `2000`.
- **Postgres `NUMERIC(19,4)` + Prisma `Decimal`** — Prisma maps this to `Decimal.js`, arithmetic stays exact. Slightly slower than integers, but reads naturally and handles currencies with different minor-unit counts if you ever go multi-country.
- Never `Float`/`Double` in the Prisma schema for money. If it's already there, that's worth fixing before volume grows.

Note the mismatch you have to handle: Lenco **returns** money as strings, but **accepts** `amount` as a JSON number in transfer requests. So you serialize out of your exact representation into a number at the request boundary, and parse strings into it on the way back. Keep both conversions in the Lenco service, not scattered through your app.

### Transfer request DTOs

Destination is "recipient ID **or** raw identifiers" — model it so the compiler enforces that you supplied one:

```typescript
class BaseTransferDto {
  @IsUUID() accountId: string;                 // account to debit

  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  amount: number;                              // major units; sent as JSON number

  @IsString()
  @Matches(/^[a-zA-Z0-9._-]+$/)
  @MaxLength(100)
  reference: string;

  @IsOptional() @IsString() @MaxLength(255) narration?: string;
}

export class TransferToMobileMoneyDto extends BaseTransferDto {
  @IsOptional() @IsUUID() transferRecipientId?: string;
  @IsOptional() @IsString() phone?: string;
  @IsOptional() @IsIn(['airtel', 'mtn', 'tnm', 'zamtel']) operator?: 'airtel' | 'mtn' | 'tnm' | 'zamtel';
  @IsOptional() @IsIn(['zm', 'mw']) country?: 'zm' | 'mw';
}

export class TransferToBankAccountDto extends BaseTransferDto {
  @IsOptional() @IsUUID() transferRecipientId?: string;
  @IsOptional() @IsString() accountNumber?: string;
  @IsOptional() @IsString() bankId?: string;
  @IsOptional() @IsString() country?: string;
}

export class TransferToOwnAccountDto extends BaseTransferDto {
  @IsUUID() creditAccountId: string;           // required — no recipient option here
}
```

class-validator can't express "exactly one of these groups" natively. Add a custom validator or a Zod refinement so a request with *neither* destination fails at the boundary rather than getting a confusing 400 back from Lenco:

```typescript
// Zod equivalent, if you prefer it for this check
const transferToMobileMoney = baseTransfer
  .extend({
    transferRecipientId: z.string().uuid().optional(),
    phone: z.string().optional(),
    operator: z.enum(['airtel', 'mtn', 'tnm', 'zamtel']).optional(),
    country: z.enum(['zm', 'mw']).optional(),
  })
  .refine((d) => !!d.transferRecipientId || (!!d.phone && !!d.operator), {
    message: 'Provide either transferRecipientId, or both phone and operator',
  });
```

### Safe transfer + recovery pattern

The important part is what happens on an ambiguous failure — see api-reference.md's "Duplicate reference" section for why:

```typescript
async transferToMobileMoney(dto: TransferToMobileMoneyDto): Promise<LencoTransfer> {
  try {
    const { data } = await firstValueFrom(
      this.http.post('/transfers/mobile-money', dto, { headers: this.authHeaders }),
    );
    if (!data.status) throw new LencoApiException(data.message, data.errorCode);
    return this.mapTransfer(data.data);
  } catch (err) {
    // Timeout / network / 5xx: we do NOT know if Lenco processed it.
    // Never blind-retry a transfer POST — reconcile by reference instead.
    if (this.isAmbiguousFailure(err)) {
      const existing = await this.findTransferByReference(dto.reference); // GET /transfers/status/:ref
      if (existing) return existing;   // it landed after all
    }
    throw err;
  }
}

async findTransferByReference(reference: string): Promise<LencoTransfer | null> {
  try {
    const { data } = await firstValueFrom(
      this.http.get(`/transfers/status/${encodeURIComponent(reference)}`, { headers: this.authHeaders }),
    );
    return data.status ? this.mapTransfer(data.data) : null;
  } catch (err) {
    if (err.response?.status === 404) return null;  // genuinely doesn't exist — safe to re-POST
    throw err;
  }
}
```

Because this endpoint is keyed on **your** reference, derive it deterministically from your own domain entity (payout ID, order ID) rather than a timestamp or random UUID generated at call time. If the reference changes between attempts, the duplicate check can't protect you and you can double-pay. That's the whole point of an idempotency key.

### List filters

```typescript
export class ListTransfersQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number = 1;
  @IsOptional() @IsDateString() from?: string;   // YYYY-MM-DD
  @IsOptional() @IsDateString() to?: string;
  @IsOptional() @IsString() search?: string;
  @IsOptional() @IsUUID() accountId?: string;
  @IsOptional() @IsUUID() transferRecipientId?: string;
  @IsOptional() @IsIn(['mobile-money', 'bank-account', 'lenco-money', 'lenco-merchant']) type?: string;
  @IsOptional() @IsIn(['pending', 'successful', 'failed']) status?: string;
  @IsOptional() @IsString() country?: string;
}
```

`status=pending` + a `from` date is the query your reconciliation job wants — it pulls only unresolved transfers from Lenco's side instead of paging the full history. Push the filter to the API, same reasoning as a `WHERE` clause instead of filtering in memory.
