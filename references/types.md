# Lenco TypeScript Types

These map Lenco's raw response shapes. In your NestJS service, map these to your own domain types at the service boundary rather than passing them straight through to controllers — see SKILL.md's "Core architecture pattern" for why.

```typescript
interface LencoAccount {
  id: string;
  details: {
    type: string;
    accountName: string;
    tillNumber?: string;
  };
  type: string;
  status: string;
  createdAt: string;
  currency: string;
  availableBalance: string | null;
  ledgerBalance: string | null;
}

// LencoTransfer moved to transfers.md — see that file for the verified shape

// LencoCollection moved to collections.md — see that file for the verified shape

interface TransferRequest {
  accountId: string;
  amount: number;
  narration?: string;
  reference: string;
  // + type-specific fields per transfer destination
}

// CollectionRequest / InitiateCollectionDto moved to collections.md
```

## Resolve types (verified against Lenco's OpenAPI spec, 2026-08-06)

```typescript
interface ResolveMobileMoneyResponse {
  type: 'mobile-money';
  accountName: string;
  phone: string;
  operator: string;
  country: string;
}

interface ResolveBankAccountResponse {
  type: 'bank-account';
  accountName: string;
  accountNumber: string;
  bank: { id: string; name: string; country: string };
}

interface ResolveLencoMoneyResponse {
  type: 'lenco-money';
  accountName: string;
  walletNumber: string;
}

interface ResolveLencoMerchantResponse {
  type: 'lenco-merchant';
  accountName: string;
  tillNumber: string;
}
```

```typescript
export class ResolveMobileMoneyDto {
  @IsString()
  phone: string;

  @IsIn(['airtel', 'mtn', 'zamtel'])
  operator: 'airtel' | 'mtn' | 'zamtel';

  @IsOptional()
  @IsIn(['zm']) // resolve endpoint currently only supports zm, unlike collections which also take mw
  country?: 'zm';
}

export class ResolveBankAccountDto {
  @IsString()
  accountNumber: string;

  @IsString()
  bankId: string; // from GET /banks?country=zm — not a public bank sort code

  @IsOptional()
  @IsString()
  country?: string;
}
```

Service pattern — same "check `status`, don't leak the raw shape" rule as everywhere else:

```typescript
async resolveMobileMoneyAccount(dto: ResolveMobileMoneyDto): Promise<ResolveMobileMoneyResponse> {
  const { data } = await firstValueFrom(
    this.http.post('/resolve/mobile-money', dto, { headers: this.authHeaders }),
  );
  if (!data.status) {
    // 400 here means "not found", not a validation error — surface that distinction to the caller
    throw new LencoApiException(data.message, data.errorCode ?? 'RESOLVE_NOT_FOUND');
  }
  return data.data;
}
```

## Transfer recipient types (verified against Lenco's OpenAPI spec, 2026-08-06)

`details` is a **discriminated union** keyed on `details.type`. Model it as one, not as a single interface with every field optional — otherwise TypeScript lets you read `recipient.details.phone` on a bank account and hand `undefined` to a transfer call, which fails at runtime instead of compile time. The discriminated union makes the compiler force a `switch` on `type` before you touch type-specific fields.

```typescript
type LencoRecipientDetails =
  | {
      type: 'bank-account';
      accountName: string;
      accountNumber: string;
      bank: { id: string; name: string; country: string };
    }
  | { type: 'mobile-money'; accountName: string; phone: string; operator: string }
  | { type: 'lenco-money'; accountName: string; walletNumber: string }
  | { type: 'lenco-merchant'; accountName: string; tillNumber: string };

interface LencoTransferRecipient {
  id: string;                              // 36-char UUID
  currency: string;                        // e.g. 'ZMW'
  type: 'bank-account' | 'mobile-money' | 'lenco-money' | 'lenco-merchant';
  country: string;                         // e.g. 'zm'
  details: LencoRecipientDetails;
}
```

Narrowing in practice — the compiler guarantees you can't reach a field that isn't there:

```typescript
function describeRecipient(r: LencoTransferRecipient): string {
  switch (r.details.type) {
    case 'bank-account':
      return `${r.details.accountName} — ${r.details.bank.name} ${r.details.accountNumber}`;
    case 'mobile-money':
      return `${r.details.accountName} — ${r.details.operator} ${r.details.phone}`;
    case 'lenco-money':
      return `${r.details.accountName} — wallet ${r.details.walletNumber}`;
    case 'lenco-merchant':
      return `${r.details.accountName} — till ${r.details.tillNumber}`;
  }
}
```

**Caveat on the list endpoint:** `GET /transfer-recipients` documents every `details` field as present-but-nullable on every record (`accountNumber: string | null`, `phone: string | null`, etc.) rather than as a clean union. The examples still only populate the fields for that record's type. Two options:

- **Recommended:** keep the union above and narrow on `details.type` — it matches the actual data and gives you exhaustiveness checking. If Lenco genuinely sends nulls for the other fields, extra nulls are harmless; you're just not reading them.
- **Defensive:** if you don't trust the shape, validate list responses at the boundary with a Zod discriminated union (`z.discriminatedUnion('type', [...])`) so a schema drift on Lenco's side fails loudly at parse time in your service rather than silently as `undefined` three layers up.

The second is worth it specifically here because recipients feed directly into money movement — a silently-undefined `phone` becomes a failed or misrouted transfer.

### Create DTOs

No `accountName` field on any of these — Lenco resolves the name from the identifiers.

```typescript
export class CreateBankRecipientDto {
  @IsString() accountNumber: string;
  @IsString() bankId: string;              // from GET /banks
  @IsOptional() @IsString() country?: string;
}

export class CreateMobileMoneyRecipientDto {
  @IsString() phone: string;
  @IsIn(['airtel', 'mtn', 'zamtel']) operator: 'airtel' | 'mtn' | 'zamtel';
  @IsOptional() @IsIn(['zm']) country?: 'zm';
}

export class CreateLencoMoneyRecipientDto {
  @IsString() walletNumber: string;
}

export class CreateLencoMerchantRecipientDto {
  @IsString() tillNumber: string;
}
```

### List query params

```typescript
export class ListRecipientsQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number = 1;

  @IsOptional()
  @IsIn(['mobile-money', 'bank-account', 'lenco-money', 'lenco-merchant'])
  type?: 'mobile-money' | 'bank-account' | 'lenco-money' | 'lenco-merchant';

  @IsOptional() @IsString() country?: string;
}
```

Filter server-side with `type`/`country` rather than pulling all pages and filtering in memory — with `perPage: 100` a merchant with a few thousand recipients means dozens of round trips and needless egress for data you throw away. Same reasoning as pushing a `WHERE` clause into SQL instead of filtering an array after `findMany()`.

## Settlement & transaction types (verified 2026-08-06)

```typescript
interface LencoSettlement {
  id: string;
  amountSettled: string;              // decimal string — amount minus fee when bearer is 'merchant'
  currency: string;
  createdAt: string;
  settledAt: string | null;
  status: 'pending' | 'settled';
  type: 'instant' | 'next-day';
  accountId: string;                  // which of your accounts was credited
  collection: LencoCollection;        // embedded; its own .settlement is always null
}

interface LencoTransaction {
  id: string;
  amount: string;                     // decimal string
  currency: string;
  narration: string;                  // e.g. "Transfer / 240730006" — embeds lencoReference
  type: 'credit' | 'debit';
  datetime: string;                   // ISO8601 UTC
  accountId: string;
  balance: string | null;             // running balance AFTER this entry
}
```

`LencoCollection` lives in `collections.md`. Note the embedded collection on a settlement omits `mobileMoneyDetails.operatorTransactionId` — model it as `Omit<>` or just treat that field as optional everywhere:

```typescript
type LencoSettlementCollection = Omit<LencoCollection, 'settlement'> & { settlement: null };
```

### List query DTOs

```typescript
export class ListSettlementsQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number = 1;
  @IsOptional() @IsDateString() from?: string;
  @IsOptional() @IsDateString() to?: string;
  @IsOptional() @IsIn(['pending', 'settled']) status?: 'pending' | 'settled';
  @IsOptional() @IsIn(['instant', 'next-day']) type?: 'instant' | 'next-day';
  @IsOptional() @IsIn(['card', 'mobile-money', 'bank-account']) collectionType?: string;
  @IsOptional() @IsString() country?: string;
}

export class ListTransactionsQueryDto {
  @IsOptional() @Type(() => Number) @IsInt() @Min(1) page?: number = 1;
  @IsOptional() @IsIn(['credit', 'debit']) type?: 'credit' | 'debit';
  @IsOptional() @IsDateString() from?: string;
  @IsOptional() @IsDateString() to?: string;
  @IsOptional() @IsString() search?: string;
  @IsOptional() @IsUUID() accountId?: string;
}
```

Note `collectionType` on settlements (not `type` — `type` there means instant/next-day). Easy to get wrong; they're different axes.
