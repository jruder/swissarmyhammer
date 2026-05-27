---
name: no-prisma
description: Enforce SpikyHed Hard Rule §1 — Drizzle is the only ORM. Block any import of Prisma, Sequelize, or TypeORM in source files.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "apps/**/*.{ts,tsx,js,mjs,cjs}"
    - "packages/**/*.{ts,tsx,js,mjs,cjs}"
    - "tools/**/*.{ts,tsx,js,mjs,cjs}"
tags:
  - spikyhed-constitution
  - orm
severity: error
timeout: 30
---

# No Prisma / Sequelize / TypeORM

SpikyHed Hard Rule §1: **ORM — Drizzle only.**

This validator blocks any imports of competing ORMs at the agent's edit/write
boundary. Catching the violation here saves a CI cycle and a code review.

## What to flag

ANY of the following in a staged or about-to-be-written file:

1. `import ... from '@prisma/client'`
2. `import ... from 'prisma'`
3. `import ... from 'sequelize'` (or `sequelize-typescript`)
4. `import ... from 'typeorm'`
5. `require('@prisma/client' | 'prisma' | 'sequelize' | 'typeorm')`
6. Type-only imports of the same packages (`import type {...}`) — still flagged.
7. A new entry in any `package.json` under `dependencies` or
   `devDependencies` adding `@prisma/client`, `prisma`, `sequelize`,
   `sequelize-typescript`, or `typeorm`.

## What NOT to flag

- `drizzle-orm`, `drizzle-kit`, `drizzle-zod` — these are the approved stack.
- Documentation that mentions Prisma by name (markdown files are not matched).
- A historical migration file that references Prisma in a comment.

## Why

The repo invariant check (`pnpm constitution:check`) already enforces this at
commit time. Validating at the agent boundary catches it earlier — before the
PR even exists — and the failure message can point the agent at the Drizzle
helpers in `packages/db` so it self-corrects.

## Remediation message

When blocking, the validator should tell the agent:

> SpikyHed Hard Rule §1 forbids non-Drizzle ORMs. Use Drizzle helpers from
> `@spikyhed/db/schema/*`. If you genuinely need an ORM Drizzle can't model
> (vanishingly rare), open a Constitution amendment PR before adding the
> dependency.
