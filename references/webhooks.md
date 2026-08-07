# Lenco Webhooks

Verified against Lenco's webhook docs 2026-08-06; setup + signing key corrected against a live production dashboard 2026-08-07.

## Setup

Configure the webhook URL in the **Lenco dashboard's webhook settings** — self-service, no support email needed. The same screen displays your **webhook signing secret**; copy it into `LENCO_WEBHOOK_SECRET`. (Lenco's docs still describe emailing support@lenco.co to register the URL — that's outdated; the dashboard is the current path. Verified on a live production account, 2026-08-07.)

Lenco's own setup notes still apply:
- If using .htaccess, include the trailing `/` on the URL you register
- The URL must be publicly reachable (localhost won't receive events)
- Test-post to it first and confirm your handler actually sees the POST body

The route must be **unauthenticated** — Lenco isn't sending your API token, it's sending a signature. Which means your auth middleware must not block it, and the signature check is the *only* thing standing between a stranger and a fake "payment successful" event. Get it right.

## Event envelope

Webhook payloads are **not** the standard API envelope. There's no `status`/`message`/`data` wrapper — it's:

```json
{ "event": "collection.successful", "data": { ... } }
```

If you reuse your API response parser here, it'll fail on the missing `status` field. Parse webhooks with their own type.

```typescript
type LencoWebhookEvent =
  | { event: 'transfer.successful';   data: LencoTransfer }
  | { event: 'transfer.failed';       data: LencoTransfer }
  | { event: 'collection.successful'; data: LencoCollection }
  | { event: 'collection.failed';     data: LencoCollection }
  | { event: 'collection.settled';    data: LencoCollection }
  | { event: 'transaction.credit';    data: LencoTransaction }
  | { event: 'transaction.debit';     data: LencoTransaction };
```

| Event | Meaning |
|---|---|
| `transfer.successful` | A transfer completed from an account linked to your API token |
| `transfer.failed` | A transfer you attempted failed |
| `collection.successful` | A collection completed — the customer's money moved |
| `collection.failed` | A collection failed |
| `collection.settled` | **Your account was credited** for a collection |
| `transaction.credit` | An account linked to your token was credited |
| `transaction.debit` | An account linked to your token was debited |

`collection.successful` and `collection.settled` are separate events for the same collection and both fire. Deliver goods on `successful`; reconcile your ledger on `settled`. See `collections.md` for why these differ.

Note `transaction.*` overlaps with the others: a successful transfer also produces a `transaction.debit`. Don't double-count if you're building a ledger — pick one event family as your source of truth.

## Signature verification

Lenco sends `X-Lenco-Signature`: an HMAC-SHA512 of the event payload, keyed by your **webhook signing secret from the dashboard** (verified in production, 2026-08-07).

> **Legacy note:** Lenco's older docs and examples derive the key as the SHA256 hex digest of your API token (`webhook_hash_key`) instead of a dashboard-issued secret. Lenco ships no changelog, so if signature verification fails with the dashboard secret on an older account, try the derived key — and prefer the dashboard secret everywhere else.

### Raw body vs re-serialized — Lenco's own examples disagree

This matters, so read it before you pick:

- Their **PHP** example hashes `file_get_contents("php://input")` — the **raw bytes** off the wire.
- Their **Node** example hashes `JSON.stringify(req.body)` — the body **parsed and re-serialized**.

These are only equivalent if your JSON serializer reproduces Lenco's bytes exactly: same key order, same whitespace, same unicode escaping, same number formatting. `JSON.stringify` preserves insertion order from the parse, so it usually works — until a payload contains a non-ASCII character, a float that round-trips differently, or Lenco changes their serializer. Then signatures start failing intermittently, and it will look like an attack rather than a bug.

**Use the raw body.** It's what the PHP example does, it's what the signature is actually computed over, and it can't drift. The Node example is a convenience shortcut that happens to work.

```typescript
import * as crypto from 'crypto';

export function verifyLencoSignature(
  rawBody: Buffer,
  signatureHeader: string | undefined,
  webhookSecret: string,   // from the Lenco dashboard's webhook settings, via env
): boolean {
  if (!signatureHeader) return false;

  const expected = crypto.createHmac('sha512', webhookSecret).update(rawBody).digest('hex');

  const a = Buffer.from(expected, 'utf8');
  const b = Buffer.from(signatureHeader, 'utf8');

  // timingSafeEqual THROWS on length mismatch — check length first, or a short
  // header crashes the guard (500) instead of cleanly rejecting (401).
  if (a.length !== b.length) return false;

  return crypto.timingSafeEqual(a, b);
}
```

**Two things this guards against:**

1. **Timing attacks.** `===` on strings short-circuits at the first differing byte, so response time leaks how much of a guessed signature was correct — enough to reconstruct it byte by byte over many requests. `timingSafeEqual` compares in constant time. (Lenco's PHP example comments that it avoids a timing attack but then uses `!==`, which doesn't. Don't copy that.)
2. **The length-mismatch crash.** `crypto.timingSafeEqual` throws a `RangeError` if the two buffers differ in length. An attacker sending `X-Lenco-Signature: x` would produce a 500 rather than a clean rejection — noisy, and a 500 tells Lenco the event was unacknowledged so they'll retry it for 24 hours. Check lengths first.

### Getting the raw body in NestJS

Express parses JSON before your handler sees it, so you must opt into keeping the raw bytes:

```typescript
// main.ts
const app = await NestFactory.create(AppModule, { rawBody: true });
```

Then `req.rawBody` is a `Buffer` on `RawBodyRequest<Request>`. Verify in a guard so a bad signature never reaches handler logic:

```typescript
@Injectable()
export class LencoWebhookGuard implements CanActivate {
  constructor(private config: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest<RawBodyRequest<Request>>();
    if (!req.rawBody) return false;   // rawBody not enabled — fail closed, don't fall back
    return verifyLencoSignature(
      req.rawBody,
      req.headers['x-lenco-signature'] as string,
      this.config.get('lenco.webhookSecret')!,
    );
  }
}
```

Failing closed when `rawBody` is missing matters: if someone later removes `rawBody: true`, you want webhooks to break loudly, not to silently skip verification.

## Responding

Respond **200, 201, or 202**. Anything else is treated as unacknowledged and Lenco retries **every 30 minutes for 24 hours**. The response body is discarded — only the status code is read.

Lenco explicitly warns that a slow handler may time out and be counted as unacknowledged. So acknowledge first, work after:

```typescript
@Controller('webhooks/lenco')
export class LencoWebhookController {
  constructor(@InjectQueue('lenco') private queue: Queue) {}

  @UseGuards(LencoWebhookGuard)
  @HttpCode(200)
  @Post()
  async handle(@Body() event: LencoWebhookEvent) {
    await this.queue.add('lenco-event', event);   // fast enqueue only
    return { received: true };
  }
}
```

**The worker must be idempotent.** Retries mean you will see the same event more than once, and a duplicate `collection.successful` that credits a wallet twice is a real financial bug. Dedupe on `data.lencoReference` + `event` (a unique constraint in Postgres is the simplest enforcement — let the DB reject the second insert rather than checking-then-inserting, which races).

## Don't rely on webhooks alone

Lenco says this themselves: if your server is down, events fire and fail while customers keep transacting. Their recommendation is a re-query service polling every ~30 minutes until a terminal status.

```typescript
// @Cron or a BullMQ repeatable job
async reconcile() {
  const pending = await this.repo.findUnresolved({ olderThan: '5m' });
  for (const tx of pending) {
    const remote = tx.kind === 'transfer'
      ? await this.lenco.getTransferByReference(tx.reference)   // GET /transfers/status/:reference
      : await this.lenco.getCollectionByReference(tx.reference); // GET /collections/status/:reference
    if (remote && remote.status !== 'pending') await this.applyStatus(tx, remote);
  }
}
```

Stop polling a record once it reaches `successful` or `failed`, and set a cutoff (24-48h) after which stuck records get flagged for human review rather than polled forever. Both endpoints 404 when the reference doesn't exist — treat that as "never landed", which is the signal that it's safe to re-initiate.

## Security checklist

- [ ] Signature verified against the **raw** body, in a guard, failing closed
- [ ] Length-checked before `timingSafeEqual`
- [ ] Route excluded from auth middleware but **not** from the signature guard
- [ ] Handler returns 200 immediately; real work is queued
- [ ] Worker dedupes on `lencoReference` — enforced by a DB constraint, not an application-level check
- [ ] Reconciliation job running independently of webhooks
- [ ] Webhook payloads logged (minus card details) — when a customer disputes, the raw event is your evidence
