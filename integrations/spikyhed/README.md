# spikyhed-constitution

A SwissArmyHammer package that encodes the SpikyHed unified-platform operating
doctrine — Hard Rules, deployment protocol, Jackalope guard, Nightly Auto-Ship
discipline — as `avp` validators and SAH skills.

## What's inside

### Skills

| Skill | Purpose |
|---|---|
| `/deploy-to-prod` | The "Deploy to prod agent protocol" from CLAUDE.md §16, executable in any MCP-compatible agent. Confirms green CI, dispatches `build.yml` with proper scope, monitors to green, applies manifests via SSH. |
| `/jackalope-dispatch` | Single-use approval protocol for self-hosted runner image builds. Touches the sentinel, runs the dispatch, and is consumed by the PreToolUse hook. |
| `/drizzle-blast-radius` | Before changing a Drizzle schema, run a call-graph blast-radius over `packages/db/schema/*` and report every consumer that needs migration. |
| `/nightly-auto-ship` | Wraps the 4:30am PT auto-ship in a `/finish` loop — open PRs that fail CI get re-implemented and re-tested before merge instead of being silently labeled `auto-ship-failed`. |

### Validators

| Validator | Hard Rule | Severity |
|---|---|---|
| `no-prisma` | §1 — Drizzle only. Blocks imports of `@prisma/client`, `prisma`, `sequelize`, `typeorm`. | error |
| `no-tailwind` | §2 — Plain CSS only. Blocks `tailwindcss` imports and `class=` strings matching Tailwind atomics. | error |
| `zod-required` | §6 — Zod for ALL input validation. Flags API routes, form handlers, and ingest endpoints that parse input without `z.<schema>.parse`. | error |
| `pino-required` | §7 — Pino structured logging. Flags `console.log`/`console.error` in `apps/*/src/` and `packages/*/src/`. | warning |
| `no-rw-ssh-bindmount` | §15 — Never bind-mount `~/.ssh` rw into a container. Greps Dockerfiles, docker-compose, Kubernetes manifests for rw bind mounts targeting `.ssh`. | error |
| `python-deps-declared` | §18 — Every Python `import` resolves to a declared dependency, stdlib, or the package itself. | error |
| `never-push-during-ci` | §14 — Pre-flight check before `git push origin main`: blocks if `gh run list` shows in-flight runs. | warning |

## Install

### Via mirdan (preferred)

```bash
mirdan install spikyhed-constitution
```

This drops the skills into `~/.sah/skills/` and registers the validators with
the installed `avp` validators directory.

### Vendor into a repo

```bash
# From a SpikyHed clone:
cp -R /path/to/swissarmyhammer/integrations/spikyhed/skills/*       .sah/skills/
cp -R /path/to/swissarmyhammer/integrations/spikyhed/validators/*   .sah/validators/
```

Project-level `.sah/` wins over user-level `~/.sah/`, so vendoring is the right
choice when the rules differ across forks.

## Wiring `avp init` into the repo

```bash
cd /path/to/spikyhed
avp init                 # Installs validator runner hooks
avp list                 # Confirm all seven validators are active
```

The validators run as PreToolUse hooks on `Edit` / `Write` tool calls and as
PreCommit hooks via the `.githooks/` machinery the SpikyHed Constitution
already installs.

## Why not encode the rules in `eslint`?

Two reasons:

1. **Several rules are non-JS** — `no-rw-ssh-bindmount` reads Dockerfiles,
   `python-deps-declared` reads `requirements.txt`, `never-push-during-ci`
   queries `gh`. ESLint can't help.
2. **Block at the agent, not at the build** — `avp` blocks the bad edit before
   it's written. ESLint catches it after the fact, during CI, after the agent
   already burned tokens producing the wrong code.

## License

MIT OR Apache-2.0
