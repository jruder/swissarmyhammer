---
name: zod-required
description: Enforce SpikyHed Hard Rule §6 — Zod for ALL input validation. Flag API routes, form handlers, and ingest endpoints that parse input without a Zod schema.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "apps/**/src/**/+server.{ts,js}"
    - "apps/**/src/**/+page.server.{ts,js}"
    - "apps/**/src/routes/api/**/*.{ts,js}"
    - "apps/captrack-api/src/**/*.{ts,js}"
    - "apps/lifelog-ingest/src/**/*.{ts,js}"
    - "apps/mobile-bff/src/**/*.{ts,js}"
    - "apps/*-worker/src/**/*.{ts,js}"
tags:
  - spikyhed-constitution
  - validation
severity: error
timeout: 60
---

# Zod Required for Input Validation

SpikyHed Hard Rule §6: **Zod for ALL input validation. Every form, API route,
and ingestion endpoint must use Zod schemas.**

## What to flag

Examine the file content for handler functions accepting external input
without Zod validation:

1. **SvelteKit endpoints** (`+server.ts`, `+page.server.ts`): a `POST`, `PUT`,
   `PATCH`, or `DELETE` handler that calls `request.json()`, `request.formData()`,
   `event.url.searchParams`, or `event.cookies.get()` and then accesses
   properties on the result WITHOUT a preceding `<Schema>.parse(...)` /
   `<Schema>.safeParse(...)` call.

2. **Node HTTP handlers** (`createServer`, Express, Fastify, Hono): a handler
   reading `req.body`, `req.query`, `req.params`, `req.headers` without
   passing the value through `z.<schema>.parse`.

3. **Queue / event consumers**: a Restate handler, Redpanda consumer, or BullMQ
   processor that destructures the message payload without parsing it.

4. **Webhook receivers**: handlers under `apps/*/src/webhooks/` or similar
   that accept a body and trust its shape.

## What NOT to flag

- Internal function calls that pass already-validated data through. The
  validator runs once at the boundary; downstream code may consume the
  resulting Zod-inferred type freely.
- GET endpoints that read no input.
- Handlers where the only input is a path parameter constrained by the
  router's own pattern (e.g. SvelteKit `[id=integer]` matcher). Path matchers
  ARE the validation in that case.

## How to check

For each handler in scope, look for:

```ts
import { z } from 'zod';                 // or 'zod/v4', '@spikyhed/core/zod'
const Body = z.object({ /* ... */ });

export const POST = async ({ request }) => {
  const body = Body.parse(await request.json());   // ← the required pattern
  // ...
};
```

Anti-patterns to flag:

```ts
// ❌ direct access to .json() result
const body = await request.json();
return doThing(body.userId);

// ❌ as-cast bypass
const body = (await request.json()) as { userId: string };

// ❌ type assertion via JSDoc
/** @type {{ userId: string }} */
const body = await request.json();
```

## Why

Untrusted input that flows into Drizzle queries or downstream services without
schema validation is the #1 SpikyHed bug class. CI's `bundle-and-contracts`
check is a backstop, but it doesn't run against draft edits — this validator
does.

## Remediation message

> SpikyHed Hard Rule §6 requires Zod on every external input. Add a schema:
>
> ```ts
> import { z } from 'zod';
> const <SchemaName> = z.object({ /* fields */ });
> const body = <SchemaName>.parse(await request.json());
> ```
>
> The repo provides shared schemas under `packages/core/src/schemas/` — check
> there first to avoid duplication.
