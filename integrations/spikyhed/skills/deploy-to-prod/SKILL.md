---
name: deploy-to-prod
description: Deploy a SpikyHed change to production. Use when the user says "deploy to prod", "ship this", "promote to prod", or any variant. Follows the CLAUDE.md §16 protocol — green-CI check, scoped image build, monitor to green, manifest apply via SSH. Self-hosted runner image rebuilds require the Jackalope token; this skill never assumes it.
license: MIT OR Apache-2.0
compatibility: Requires `gh` (GitHub CLI) on PATH, SSH access to the prod host configured as `prod`, and the `shell` MCP tool. Operates on the SpikyHed unified-platform monorepo layout.
metadata:
  author: spikyhed
  version: "1.0.0"
---

# Deploy to Prod

Executes the SpikyHed "Deploy to prod" agent protocol from CLAUDE.md §16. Every
step is gated; no step is skipped because the next one looked safe in isolation.

## When to use

The user explicitly asks to deploy. Convenience phrases like "ship this" count
*only* for application image builds and manifest applies — they do NOT
authorize self-hosted runner rebuilds (the `build.yml` workflow). Those require
the literal word `jackalope` in the user's prompt.

## Protocol

1. **Confirm green CI on `main`.** Run `gh run list --limit 5` and verify no
   runs against `HEAD` are `in_progress` or `queued`, and the most recent
   workflows for `HEAD` are `success`. If anything is in flight, STOP and tell
   the user — never push during active CI (Hard Rule §14).

2. **Decide scope.** Read the user's prompt:
   - Default: targeted image build (`scope=targeted`).
   - `"rebuild full"` → `scope=full`.
   - `"manifests only"` → skip the build step entirely.

3. **Dispatch the build** (skip if manifests-only):
   ```bash
   gh workflow run build.yml --ref main \
     -f scope=<targeted|full> \
     -f reason="<user-supplied reason, ≥10 chars>"
   ```
   The `reason` MUST be specific to this deploy. "Deploy" is not a reason; "Roll
   out APNS retry fix from JCR-187" is.

4. **Monitor the run to green.**
   ```bash
   gh run list --workflow=build.yml --limit 1
   gh run watch <run-id>
   ```
   On flake (network errors, transient infra), `gh run rerun --failed`.
   On real failures, STOP and report — don't rerun broken builds.

5. **Apply the manifests.** For each changed workload under `infra/k8s/`:
   ```bash
   cat infra/k8s/<workload>/<file>.yaml | ssh prod 'sudo kubectl apply -f -'
   ```

6. **Verify.** `ssh prod 'sudo kubectl get pods -A | grep -v Running | grep -v Completed'`
   should return empty (or only deliberately-stopped workloads).

7. **Report.** Summarize: workflows dispatched, run IDs, image digests applied,
   pod state. Link to the GitHub run.

## Self-hosted runner rebuild (Jackalope)

If the deploy requires rebuilding the runner image itself, you MUST NOT proceed
without the user saying `jackalope` in their prompt. Use the `/jackalope-dispatch`
skill instead.

## Hard refusals

- **Do not push to main.** This skill never runs `git push`. If the deploy
  requires unpushed commits, STOP and ask the user to push first via a PR.
- **Do not bypass the Constitution.** If `pnpm constitution:check` fails on
  HEAD, STOP — fix or label `wip` before deploying.
- **Never use `--no-verify`** on any git operation.

## Examples

### Example 1: targeted deploy of a single API change

User says: "deploy to prod — APNS retry fix JCR-187"

Actions:
1. `gh run list --limit 5` → most recent run for `HEAD` is success, nothing
   in flight. ✓
2. `gh workflow run build.yml --ref main -f scope=targeted -f reason="Deploy APNS retry fix JCR-187"`.
3. `gh run watch <id>` → green after 8 min.
4. `cat infra/k8s/apns-sender/deployment.yaml | ssh prod 'sudo kubectl apply -f -'`.
5. `ssh prod 'sudo kubectl get pods -n apns-sender'` → all `Running`.
6. Report: build run `gh run view 12345`, deploy applied, no off-state pods.

### Example 2: manifests-only deploy of a config change

User says: "deploy to prod, manifests only — bump SENTRY_TRACES_SAMPLE_RATE"

Actions:
1. CI check (still required). ✓
2. Skip step 3-4 (no image build needed).
3. `cat infra/k8s/depmap/configmap.yaml | ssh prod 'sudo kubectl apply -f -'`.
4. `ssh prod 'sudo kubectl rollout restart deployment/depmap -n depmap'`.
5. Verify pods come up green.

## Troubleshooting

### `gh run list` shows a build in progress on `HEAD`

- **Cause**: someone else dispatched, or auto-ship is mid-run.
- **Solution**: STOP. Wait for the run to finish; never trigger concurrent builds.

### `kubectl apply` fails with `the server doesn't have a resource type`

- **Cause**: CRD missing on the cluster — manifest references a kind the cluster
  doesn't know about.
- **Solution**: STOP. Apply the CRD first (separate review), then retry. Never
  `kubectl apply --force`.

### Trivy image scan fails on CRITICAL

- **Cause**: the new image has a critical CVE.
- **Solution**: STOP. Fix the vulnerability or add a precise pin to
  `.trivyignore` with justification. Do NOT broaden the ignore.
