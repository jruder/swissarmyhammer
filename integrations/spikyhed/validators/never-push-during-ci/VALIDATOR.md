---
name: never-push-during-ci
description: Enforce SpikyHed Hard Rule §14 — before `git push origin main`, verify no GitHub Actions runs against HEAD are in progress or queued. Prevents the concurrency-group cancellation that killed a 40-minute Docker build with a docs-only push.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  tools:
    - Bash
tags:
  - spikyhed-constitution
  - ci-safety
severity: warning
timeout: 30
---

# Never Push to Main During Active CI

SpikyHed Hard Rule §14: **Before every `git push origin main`, run
`gh run list --limit 5` and confirm NO runs are `in_progress` or `queued`.
If a build is running, push to a branch and PR instead.**

GitHub Actions concurrency groups cancel in-flight jobs — a 40-minute Docker
build will be killed by a docs-only push.

## What to flag

The validator runs as a PreToolUse hook on `Bash` calls. Inspect the
`command` argument:

1. **Direct push to main**:
   ```
   git push origin main
   git push --set-upstream origin main
   git push -u origin main
   git push origin HEAD:main
   git push origin main:main
   ```

2. **Force pushes to main** (always blocked anyway, but this hook catches
   them as a special case):
   ```
   git push --force origin main
   git push -f origin main
   git push --force-with-lease origin main
   ```

3. **Push without explicit refspec** when the current branch is `main`:
   ```
   git push                   # only flagged if HEAD == main
   git push origin            # ditto
   ```

For matches, the validator must:

1. Run `gh run list --limit 5 --json status,conclusion,workflowName,databaseId,headSha`.
2. Filter to runs whose `headSha` matches `git rev-parse HEAD`.
3. If any have `status` in `{queued, in_progress, waiting, requested, pending}`,
   BLOCK the push with the message below.
4. Otherwise allow.

## What NOT to flag

- Pushes to feature branches.
- Pushes to `gh-pages`, `docs-*`, `release-*` branches.
- Dry-run pushes (`git push --dry-run`).
- Pushes inside a sub-shell that's clearly part of a test harness (heuristic:
  `command` includes `bats`, `pytest`, or `vitest`).

## Why warning, not error

The hook can be wrong: a queued run might be unrelated, or the user genuinely
needs to push to break a stuck state (rare, but real). A warning lets the user
confirm; an error blocks legitimate operator work. The hook prints the run
list so the user can decide.

## Remediation message

> SpikyHed Hard Rule §14: a push to `main` is unsafe while CI is active on
> the current SHA. In-flight runs:
>
> ```
> <gh run list output filtered to HEAD>
> ```
>
> Either:
>
> 1. Wait for the runs to complete (`gh run watch <id>`), or
> 2. Push to a feature branch and open a PR.
>
> Force-pushes to main are forbidden by another rule — don't reach for
> `--force` to "fix" this warning.
