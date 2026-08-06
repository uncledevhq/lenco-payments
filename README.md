# lenco-payments

An [Agent Skill](https://agentskills.io) for integrating the **Lenco payments API** (Zambia / Malawi mobile money, bank transfers, cards) into a backend — with the gotchas already written down.

Works with Claude Code, Claude, Cursor, Codex, Gemini CLI, and anything else that speaks the agentskills.io `SKILL.md` standard.

By **Chomba Chanda** ([uncledevhq](https://github.com/uncledevhq)) · MIT

> **Unofficial — but it works.** 🙂
> Not affiliated with, endorsed by, or reviewed by Lenco. This is a community-built skill by a developer who ships on their API. Every schema in here is transcribed from Lenco's own published OpenAPI spec and dated, so you can see how fresh it is. For anything that moves real money, test it in sandbox first — same as you would with any integration you didn't write yourself.

## Why

Your AI agent doesn't know Lenco. A Zambian payments API is barely represented in any model's training data — so today, integrating Lenco with an agent means *you* are the context pipeline: tab over to the docs, hunt down the endpoint to initiate a mobile money charge, paste it into the prompt, correct the path it hallucinated anyway, go back for the response schema, again for the webhook format. Or you let the agent crawl the docs itself and burn time and tokens rediscovering what someone already knows.

This skill ends that loop. Install it once and your agent already has the whole API in its head — every v2.0 endpoint, request/response schema, webhook signature scheme, and sandbox test number, verified against Lenco's OpenAPI spec and dated. Say "add mobile money checkout" and it puts the pieces together. No tab-switching, no doc-pasting, no hallucinated endpoints.

And it carries the layer the docs don't have — the production behavior you only learn by shipping real money:

- The duplicate-reference semantics that make a naive retry-after-timeout report "failed" on a transfer that actually went through
- Why `status` and `settlementStatus` on a collection drive different sides of your accounting, and conflating them breaks reconciliation
- Which of Lenco's own webhook signature examples to trust (they disagree), and the `timingSafeEqual` crash the obvious implementation hits
- The response envelope and error-shape inconsistencies your error handling has to survive
- How to handle money that arrives as decimal strings but is accepted as JSON numbers

Whether you've shipped five payment integrations or you're vibe-coding your first, the outcome is the same: your agent builds it the way someone who's shipped this before would — first time, not after code review caught it.

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

**With the skills CLI:**
```bash
npx skills add uncledevhq/lenco-payments
```

**Manually (Claude Code):**
```bash
git clone https://github.com/uncledevhq/lenco-payments.git \
  .claude/skills/lenco-payments
```

**Claude (web/desktop):** download this repo as a `.zip` (Code → Download ZIP) and upload it under Settings → Capabilities → Skills.

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
