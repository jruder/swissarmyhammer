---
name: nightly-auto-ship
description: Wraps the SpikyHed 4:30am PT Nightly Auto-Ship in a self-healing /finish loop. Use when the user says "run auto-ship", "do the nightly merge", or when the scheduled trigger `trig_01QRy7JSRzcfVoWNtt6uek2T` invokes it. Iterates over candidate PRs; failing PRs get one /finish round before being labeled `auto-ship-failed`, instead of being labeled immediately.
license: MIT OR Apache-2.0
compatibility: Requires `gh`, the SpikyHed CI/build/deploy workflows, the `kanban` and `code_context` MCP tools, and a working `/finish` skill in the same install. Authorized via the scheduled trigger's prompt (exempt from the Jackalope rule).
metadata:
  author: spikyhed
  version: "1.0.0"
---

# Nightly Auto-Ship

The legacy auto-ship cron runs at 4:30am PT: enumerate open PRs, exclude any
with safety labels (`wip`, `broken`, `hold`, `insecure`, `do-not-ship`,
`auto-ship-failed`), merge, build, deploy. Failures get labeled
`auto-ship-failed` and silently dropped.

This skill replaces "label and drop" with "one round of /finish, then label and
drop". One transient CI flake or one trivial type error no longer needs a human
in the morning.

## When to use

- Scheduled trigger `trig_01QRy7JSRzcfVoWNtt6uek2T` fires the prompt.
- A human runs `/nightly-auto-ship` manually to dry-run the flow.
- A human is debugging a previously-failed auto-ship and wants the loop to
  re-evaluate.

## Protocol

1. **Enumerate candidates.** `gh pr list --base main --state open --json number,title,labels,headRefName,mergeable,statusCheckRollup`.

2. **Filter out unsafe PRs.** Drop any with labels in
   `{wip, broken, hold, insecure, do-not-ship, auto-ship-failed}`. Also drop
   anything `mergeable` is `CONFLICTING` — rebase is out of scope.

3. **For each remaining PR**:

   a. **CI green?** Inspect `statusCheckRollup`. If all checks are
      `SUCCESS` or `NEUTRAL`, queue for merge — skip to step (d).

   b. **CI failed or in flight?** Investigate.
      - In flight → skip; let it finish on its own.
      - Failed with infra/network flake (look for known error signatures via
        `code_context search history` against the SAH shell history) → `gh run
        rerun --failed <run-id>`. Wait for completion.
      - Failed with real test/type/lint error → run `/finish` against the PR
        branch with a single objective: "make CI green on this PR". One round
        only — `/finish` may use up to its default iteration budget. Push the
        fix to the PR branch.

   c. **Re-check CI.** If green, queue for merge. If still red after one
      `/finish` round, label `auto-ship-failed` with a comment summarizing
      what was tried, and move on. Do NOT escalate to multiple rounds.

   d. **Merge.** `gh pr merge <number> --merge --auto`.

4. **Build + deploy.** After all merges, invoke `/deploy-to-prod` with
   `scope=targeted` and reason `Nightly Auto-Ship <date> — PRs <list>`.

5. **Report.** Slack-style summary:
   ```
   Nightly Auto-Ship 2026-05-18
   - 4 PRs eligible
   - 3 merged clean (#123, #124, #127)
   - 1 healed via /finish (#125, CI was green after 1 round)
   - 0 labeled auto-ship-failed
   - Deploy: build #4567 green, manifests applied to 3 workloads
   ```

## Hard constraints

- **Only one /finish round per PR.** The skill is not a babysitter — if a PR
  needs two rounds of fixes, a human needs to look at it.
- **Never override safety labels.** A `wip` PR is `wip` even if it would
  otherwise pass. Labels are author intent.
- **Never disable validators.** If `avp` blocks a fix, the PR is not eligible
  — label `auto-ship-failed` and continue.
- **Respect the Jackalope rule.** If a healed PR's deploy would require
  rebuilding the self-hosted runner, label `auto-ship-failed` with reason
  "requires Jackalope approval" and continue without rebuilding.

## Example

Scheduled trigger fires at 4:30am PT.

Actions:
1. `gh pr list ...` → 5 open PRs against main.
2. Filter: PR #130 is `wip`, drop. PR #131 has `CONFLICTING` merge state, drop.
   Three remain: #123, #125, #128.
3. #123: CI green, merge.
4. #125: CI red, failure is `TypeError: cannot read property 'id' of undefined` in `apps/captrack-ui/src/lib/foo.test.ts`. Run `/finish` against the branch; agent fixes the test by adding a guard, pushes. Re-check → green. Merge.
5. #128: CI red, failure is a real type error in cross-package contract. `/finish` round produces a fix that fails `avp` (cross-boundary type leak). Label `auto-ship-failed`, comment "validator blocked the type leak fix; needs human design call".
6. Deploy build → green. Manifests applied to depmap + captrack-ui.
7. Report posted to Slack channel `#auto-ship`.

## Troubleshooting

### `/finish` loop won't terminate

- **Cause**: PR is in a state where every fix breaks something else (flapping
  test, environment-dependent assertion).
- **Solution**: Hit the iteration budget, then label `auto-ship-failed`. Do
  NOT raise the budget per-PR.

### Build dispatched without a paired deploy

- **Cause**: image built but step 4's deploy never ran (skill aborted between
  build and deploy).
- **Solution**: `gh workflow run deploy-targeted.yml` against the now-built
  digests. NEVER assume a build implies a deploy — they're separate gates.
