---
name: pino-required
description: Enforce SpikyHed Hard Rule §7 — Pino structured JSON logging. Flag console.log / console.error / console.warn / console.info in service and API code.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "apps/**/src/**/*.{ts,tsx,js,mjs,cjs}"
    - "packages/**/src/**/*.{ts,tsx,js,mjs,cjs}"
tags:
  - spikyhed-constitution
  - logging
severity: warning
timeout: 30
---

# Pino Required for Structured Logging

SpikyHed Hard Rule §7: **Pino for structured JSON logging. Every service file
and API route must use the app's logger instance.**

## What to flag

1. `console.log(...)`, `console.error(...)`, `console.warn(...)`,
   `console.info(...)`, `console.debug(...)` in any TS/JS file under
   `apps/*/src/` or `packages/*/src/`.
2. `process.stdout.write(...)` / `process.stderr.write(...)` used as a
   logging primitive (string arguments, not stream piping).
3. Custom one-off `function log(msg) { ... }` helpers that wrap `console.*`.

## What NOT to flag

- **Test files**: `*.test.ts`, `*.spec.ts`, `tests/**/*.ts`. Tests may use
  `console.*` for debug output that vitest/jest already captures.
- **CLI binaries**: `apps/*/bin/*.ts` where stdout IS the contract (the script
  prints JSON to a piped consumer). Bin scripts get a pass.
- **Scaffolding scripts**: anything under `scripts/` or `tools/*/scripts/`.
- **Files with no other code** (one-line scripts, examples in fixtures).

## Expected pattern

Every app exports `logger` from a single file (`src/logger.ts`):

```ts
import pino from 'pino';
export const logger = pino({
  name: '<app-name>',
  level: process.env.LOG_LEVEL ?? 'info',
});
```

Other files in the app import it:

```ts
import { logger } from './logger.js';
logger.info({ userId, requestId }, 'received request');
logger.error({ err, userId }, 'failed to handle request');
```

## Why warning, not error

Some files legitimately have one stray `console.log` left over from a debug
session that the agent forgot to remove. A warning surfaces it without blocking
the edit; a follow-up `/review` round can clean them up. Hard Rule §7 is
strict at CI time — the warning here is an early reminder, not the final gate.

## Remediation message

> SpikyHed Hard Rule §7 requires Pino for structured logging. Replace
> `console.<level>(msg)` with:
>
> ```ts
> import { logger } from './logger.js';   // (or your app's logger path)
> logger.<level>({ /* structured fields */ }, msg);
> ```
>
> If you don't have a logger yet, create `src/logger.ts` following the pattern
> in `apps/captrack-api/src/logger.ts`.
