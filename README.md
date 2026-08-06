# lenco-payments

An [Agent Skill](https://agentskills.io) for integrating the **Lenco payments API** (Zambia / Malawi mobile money, bank transfers, cards) into a backend — with the gotchas already written down.

Works with Claude Code, Claude, Cursor, Codex, Gemini CLI, and anything else that speaks the agentskills.io `SKILL.md` standard.

By **Chomba Chanda** ([uncledevhq](https://github.com/uncledevhq)) · MIT

> **Unofficial — but it works.** 🙂
> Not affiliated with, endorsed by, or reviewed by Lenco. This is a community-built skill by a developer who ships on their API. Every schema in here is transcribed from Lenco's own published OpenAPI spec and dated, so you can see how fresh it is. For anything that moves real money, test it in sandbox first — same as you would with any integration you didn't write yourself.

## Why

Lenco's API docs are complete but flat. If you're building against them, you end up rediscovering the same things everyone else does:

- A `200` can carry `status: false` — the HTTP code isn't the success signal
- The shape of `data` on error is inconsistent across endpoints (`null` here, `[]` there)
- Duplicate references are rejected, and the response **doesn't** return the original transaction — so a naive retry after a timeout can report "failed" on a transfer that actually went through
- `status` and `settlementStatus` on a collection mean different things; conflating them breaks reconciliation
- Lenco's own webhook examples disagree on whether the signature covers the raw body or re-serialized JSON
- Money arrives as decimal strings but is accepted as JSON numbers

This skill encodes all of that, so your agent gets it right the first time instead of you catching it in code review — or in production.

## What's in it

| File | Covers |
|---|---|
| `SKILL.md` | Architecture, config, idempotency, security checklist |
| `references/api-reference.md` | Every v2.0 endpoint, verified request/response shapes, error codes |
| `references/transfers.md` | Outbound money: types, DTOs, decimal handling, safe retry/recovery |
| `references/collections.md` | Inbound money: mobile money, card/3DS, settlement semantics |
| `references/webhooks.md` | Signature verification, event catalogue, reconciliation |
| `references/types.md` | Accounts, resolve, recipients, settlements, transactions |
| `references/testing.md` | Full sandbox test matrix + the tests actually worth writing |

Every schema is verified against Lenco's published OpenAPI spec, with dates noted in each section.

## Install

**With the GitHub CLI:**
```bash
gh skill install <your-username>/lenco-payments
```

**Manually (Claude Code):**
```bash
git clone https://github.com/<your-username>/lenco-payments.git \
  .claude/skills/lenco-payments
```

**Claude (web/desktop):** upload the packaged `.skill` file from Releases.

Then just ask for Lenco work normally — it auto-triggers on mentions of Lenco, mobile money, ZMW collections, and similar. You don't invoke it by name.

## Code examples

The examples target **NestJS + Prisma**, since that's what it was built against. The API facts, request/response shapes, and failure semantics are framework-agnostic — the patterns translate to Express, Fastify, Django, Laravel, or whatever you're on.

## Scope and status

Covers Lenco API **v2.0** (payments: collections, transfers, recipients, settlements). Lenco also publishes a v1.0 surface (virtual accounts, bill payments, POS terminals) which is **not** covered here — it's a different product, not an older version of the same one.

Lenco ships no changelog, so their API can drift silently. Each file carries a last-verified date — treat it as a confidence marker, not a guarantee, and spot-check against the live docs before relying on anything for production money movement.

If you find drift, please open an issue (see below). That's what keeps this accurate for everyone else.

## Contributing

Found a gap or a drift? Open an issue with the endpoint and the actual payload you got back. Corrections backed by a real request/response are the most useful thing you can send.

## Author

**Chomba Chanda** (uncledev) — fullstack dev shipping production fintech in Zambia.

- GitHub: [@uncledevhq](https://github.com/uncledevhq)
- X/Twitter: [@uncledevhq](https://x.com/uncledevhq)
- LinkedIn: [uncledevhq](https://www.linkedin.com/in/uncledevhq/)
- Email: [uncledevhq@gmail.com](mailto:uncledevhq@gmail.com)

Stuck on a Lenco integration, or want a second pair of eyes before you move real money? This is the work I do — hit me up. Same goes for other work and collabs.

## License

MIT
