---
name: drizzle-blast-radius
description: Before changing a Drizzle schema, enumerate every consumer that will need to migrate. Use whenever the user wants to alter, drop, rename, or add a NOT NULL column to a table defined under `packages/db/schema/*`. Produces a precise list of files and call sites, not a guess.
license: MIT OR Apache-2.0
compatibility: Requires the `code_context` MCP tool (tree-sitter symbol search + call graph). Operates on the SpikyHed monorepo layout where shared schemas live in `packages/db/schema/` and consumers import from `@spikyhed/db/schema/*`.
metadata:
  author: spikyhed
  version: "1.0.0"
---

# Drizzle Blast Radius

A Drizzle table touches every app that imports its schema. Renaming a column or
adding a NOT NULL field without migrating consumers ships a runtime crash. This
skill enumerates the blast radius BEFORE the schema change lands.

## When to use

The user wants to:
- Rename a column or table in `packages/db/schema/*`.
- Drop a column or table.
- Add a NOT NULL column (existing rows need a backfill default OR the consumers
  need to supply the value on insert).
- Change a column's TypeScript type (e.g. `text` → `varchar(255)`, nullable →
  non-nullable).

Trivial additions (a new nullable column, a new index) don't need this skill —
they're additive and don't break consumers.

## Protocol

1. **Identify the symbol.** Run `code_context` `op: "search symbol"` for the
   table name and the column name:
   ```
   {"op": "search symbol", "query": "<tableName>"}
   {"op": "search symbol", "query": "<columnName>"}
   ```

2. **Get the inbound call graph.** For each export of the schema file:
   ```
   {"op": "get callgraph", "symbol": "<tableName>", "direction": "in", "max_hops": 3}
   ```
   The result is every file that imports the table directly OR transitively
   through a query helper.

3. **Get the file-level blast radius.** Treat the schema file as the seed:
   ```
   {"op": "get blastradius", "file_path": "packages/db/schema/<file>.ts", "max_hops": 4}
   ```

4. **Cross-check for raw SQL.** `code_context` won't catch raw SQL strings.
   Grep across apps for the table name and column name as string literals:
   ```bash
   rg -n --type ts --type sql "['\"]<tableName>['\"]|['\"]<columnName>['\"]" apps packages
   ```

5. **Cross-check Drizzle migrations.** Existing migrations under
   `packages/db/drizzle/` may reference the column. Renaming requires a paired
   migration.

6. **Produce the report.** Group results by app:
   ```
   ### Consumers of `<table>.<column>`

   - apps/depmap/src/services/foo.ts:123 — direct select
   - apps/captrack-api/src/routes/bar.ts:45 — insert (needs value supplied)
   - packages/core/src/types/dep.ts:12 — type re-export
   - apps/watcher/src/cron/health.ts:78 — RAW SQL match

   ### Migrations referencing the column

   - packages/db/drizzle/0042_initial.sql — CREATE
   - packages/db/drizzle/0089_index.sql — INDEX
   ```

7. **Recommend the migration plan.** Based on the change type:
   - **Rename**: emit `ALTER TABLE ... RENAME COLUMN`, then update all
     consumers in the same PR, then a follow-up that drops the column shim.
   - **Add NOT NULL**: emit `ALTER TABLE ... ADD COLUMN ... NULL`, backfill,
     `ALTER ... SET NOT NULL`, in three migrations.
   - **Drop**: update consumers to stop reading the column FIRST in a separate
     PR, then drop.

8. **STOP and confirm with the user** before generating the migration file.
   Migration indices are partitioned (see `MIGRATION-INDEX-BLOCKS.md`); ask
   what block to use.

## Hard refusals

- **No `DROP COLUMN` if any consumer is found in step 4.** The skill returns a
  list of consumers; it does not generate the destructive migration until the
  user explicitly accepts the list.
- **No raw SQL in `apps/*` for the touched table** without flagging it for
  review. Grep matches in step 4 are escalations, not noise.
- **No skipping step 5.** The migration directory IS authoritative for ordering.

## Example

User says: "rename `decision_ledger.author_id` to `created_by_user_id`"

Actions:
1. `code_context search symbol decision_ledger` → finds the schema.
2. `code_context get callgraph decisionLedger in` → 6 files in `apps/captrack-api`, `apps/captrack-ui`, `apps/decision-ledger`.
3. `code_context get blastradius packages/db/schema/captrack.ts in` → same 6 plus `packages/core/src/types/ledger.ts`.
4. `rg -n author_id apps packages` → 2 raw SQL matches in `apps/watcher/src/cron/audit.ts`.
5. Migration index check → next block 50-59 is unclaimed.
6. Report: 7 TS files, 2 raw SQL sites, 1 migration index pair (rename + shim).
7. Ask user: "Use migration block 50-59 for this rename pair?"
