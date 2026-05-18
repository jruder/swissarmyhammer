---
name: jackalope-dispatch
description: Single-use approval flow for dispatching or rerunning the SpikyHed self-hosted runner image build workflow (`build.yml`). Use ONLY when the user's prompt contains the literal word "jackalope". Touches the approval sentinel, runs the dispatch, and lets the PreToolUse hook consume the sentinel.
license: MIT OR Apache-2.0
compatibility: Requires `gh` on PATH, the `~/.claude/hooks/jackalope-guard.sh` PreToolUse hook installed, and write access to `/tmp/`.
metadata:
  author: spikyhed
  version: "1.0.0"
---

# Jackalope Dispatch

The Jackalope Rule (CLAUDE.md §16) is a hard guardrail: Claude must not
dispatch or rerun `build.yml` unless the user's prompt contains the literal
word `jackalope`. Every dispatch is single-use — re-approval required each
time.

## When to use

The user said `jackalope` in this turn AND wants to dispatch or rerun
`build.yml`. No other phrase (`"deploy"`, `"ship"`, `"rerun"`) satisfies the
rule.

If the user wants to deploy but hasn't said `jackalope`, use `/deploy-to-prod`
which assumes the runner image is already built.

## Protocol

1. **Verify the trigger word.** Confirm the user's most recent message contains
   the literal token `jackalope` (case-sensitive). If not, STOP and ask them
   to re-state with `jackalope`.

2. **Touch the sentinel.**
   ```bash
   touch /tmp/claude-jackalope-approved
   ```
   The PreToolUse hook reads this file's mtime to decide whether to allow the
   next `gh workflow run build.yml` / `gh run rerun` call. Sentinel is valid
   for 300 s.

3. **Run the dispatch (or rerun).** Exactly one of:
   ```bash
   gh workflow run build.yml --ref main -f scope=<scope> -f reason="<reason>"
   # or:
   gh run rerun <run-id>                  # full rerun
   gh run rerun <run-id> --failed         # only failed jobs
   ```
   The hook consumes the sentinel after this single call.

4. **Verify consumption.**
   ```bash
   test -f /tmp/claude-jackalope-approved && \
     echo "WARN: sentinel still present — hook may not have fired"
   ```
   If the sentinel survived the call, the hook is misconfigured. Tell the user.

5. **Monitor the run.** Same as `/deploy-to-prod` step 4.

## Hard refusals

- **No batched dispatches.** One sentinel, one dispatch. To do a second
  dispatch, the user must say `jackalope` again.
- **No silent retries.** If the dispatch fails (e.g. workflow not found, bad
  input), STOP — do NOT re-touch the sentinel and retry. Report to the user.
- **The scheduled Nightly Auto-Ship trigger is exempt** from this skill — its
  own prompt carries the authorization. Don't touch the sentinel on its behalf.

## Recovery: accidental dispatch without `jackalope`

If Claude already dispatched `build.yml` without the trigger word:

1. `gh run list --workflow=build.yml --limit 3`
2. `gh run cancel <databaseId>` on any queued/in-progress run.
3. Tell the user immediately, show the cancelled run IDs, and ask for explicit
   permission before any retry.

## Example

User says: "jackalope — rebuild the runner image, the base needs the new openssl"

Actions:
1. Trigger word present. ✓
2. `touch /tmp/claude-jackalope-approved`.
3. `gh workflow run build.yml --ref main -f scope=full -f reason="Rebuild runner image with new openssl base — jackalope-approved"`.
4. `test -f /tmp/claude-jackalope-approved` → file gone, hook consumed it. ✓
5. `gh run watch <id>` → green.
6. Report: dispatched run `<url>`, sentinel consumed, image now in GHCR.
