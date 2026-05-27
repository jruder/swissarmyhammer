---
name: boundary-needs-adr
description: Every new row in `BOUNDARIES.md` must have a paired ADR under `docs/decisions/`. Block edits that append a row without a corresponding ADR file.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "BOUNDARIES.md"
    - "**/BOUNDARIES.md"
tags:
  - boundary-first
  - documentation
severity: error
timeout: 30
---

# Every Boundary Has an ADR

Boundary-First doctrine treats each boundary as a decision with tradeoffs.
The decision MUST be recorded as an ADR under `docs/decisions/`. Skipping the
ADR is how doctrine erodes — six months later nobody remembers why
DeepFilterNet was abandoned for RNNoise.

## What to flag

On every edit to `BOUNDARIES.md`:

1. **Parse the table** of boundaries. Each row has at least: index, name,
   vendor, capability, swap risk, wrapped state, alternatives, last reviewed.

2. **Compute the diff** versus the file on disk (or, if no version on disk
   yet, against an empty table).

3. **For each NEW row** (added by this edit):

   a. Try to find an ADR that references this boundary. Heuristics:
      - Filename contains the protocol slug (e.g. `0007-layered-recording.md`
        for a `LayeredRecorder` port).
      - Filename contains the vendor name (`coreml`, `rnnoise`, etc.).
      - The ADR body text mentions the boundary's name or the wrapped
        protocol's name.

   b. If no ADR is found, BLOCK the edit. Message lists the new row and
      suggests an ADR filename.

4. **For each MODIFIED row** whose `wrapped state` changed from `❌`/`🔜` to
   `✅`: same check — an ADR must exist that documents the wrap decision.

5. **Last-reviewed staleness**: if any row's `Last reviewed` date is older
   than 120 days, emit a WARNING (not a block) listing the stale rows. The
   quarterly review cadence is a soft rule.

## What NOT to flag

- **Cosmetic edits** (typo fix, column alignment) that don't change the row
  count or the wrapped state. Compute a normalized hash and skip if
  unchanged.
- **Row deletions**: ADR remains as historical record. The validator
  doesn't enforce that a deletion has its own "superseded" ADR.

## Why error

The Operating Rule 7 mandates quarterly boundary review. Without ADRs that
review degenerates into "looks fine I guess". Block.

## Remediation message

> `BOUNDARIES.md` adds row #<N> for `<boundary>` with no matching ADR under
> `docs/decisions/`. Create the ADR before appending the row:
>
> ```
> docs/decisions/<NNNN>-<slug>.md
> ```
>
> Use `/new-boundary` to scaffold port + adapter + ADR + the registry row
> in one workflow. If the ADR exists but the validator can't find it,
> rename it so the filename slug matches the protocol name or vendor.
