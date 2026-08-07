# Changelog

Lenco ships no changelog for their API — this skill does. Every content correction lands here with a date and how it was verified, so you can judge freshness without diffing files.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versions are dated; "verified in production" means confirmed against a live Lenco account, which outranks "transcribed from docs."

## [1.2.0] — 2026-08-07

### Added
- `references/webhooks.md`: raw-body recipes beyond NestJS — Supabase Edge Functions/Deno, Express, Next.js App Router, Fastify — plus the Supabase `verify_jwt` trap that silently 401s Lenco's webhooks before your handler runs.

## [1.1.0] — 2026-08-07

### Corrected (verified in production)
- **Webhook setup is self-service** — configure the webhook URL in the Lenco dashboard's webhook settings. Previously (per Lenco's docs, still unchanged there): email support@lenco.co to register the URL.
- **Webhook signing key is a dashboard-issued secret** — copy it from the dashboard into `LENCO_WEBHOOK_SECRET`. Previously (and still in Lenco's docs and code examples): derive it as the SHA256 hex digest of your API token. The derived-key method is kept in `references/webhooks.md` as a legacy fallback note in case older accounts still use it.

### Changed
- `lenco.config.ts` example: `webhookHashKey` (derived) replaced with `webhookSecret` (env-provided); startup validation note now covers `LENCO_WEBHOOK_SECRET`.
- `verifyLencoSignature()` now takes the webhook secret directly instead of the API token.

## [1.0.0] — 2026-08-06

### Added
- Initial release, verified against Lenco's published OpenAPI spec (v2.0).
- `SKILL.md` — NestJS module architecture, config, idempotency semantics, verification flow, security checklist, escalation path.
- `references/api-reference.md` — every v2.0 endpoint with verified request/response shapes and error codes.
- `references/transfers.md` — transfer types, money-as-strings handling, safe retry/recovery pattern.
- `references/collections.md` — collection types, `status` vs `settlementStatus` semantics, card/3DS flow.
- `references/webhooks.md` — event envelope, signature verification (raw body), reconciliation pattern.
- `references/types.md` — accounts, resolve, recipients, settlements, transactions.
- `references/testing.md` — sandbox test matrix and the webhook tests worth writing.
