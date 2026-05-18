# boundary-first

A SwissArmyHammer package that turns the **Boundary-First Architecture**
doctrine into an executable workflow plus four `avp` validators that enforce
the doctrine at the agent's edit boundary.

Originally derived from `jruder/iphone-app-2` (Voice Memos Clone), the package
generalises to any Swift / iOS codebase that uses Ports/Adapters/Services
layering and an ADR-driven decision log.

## What's inside

### Skills

| Skill | Purpose |
|---|---|
| `/new-boundary` | The "Before writing new code" checklist from the doctrine, executable. Scans `BOUNDARIES.md`, checks `CAPABILITIES.md`, scaffolds the protocol in `Audio/Ports/`, the adapter in `Audio/Adapters/`, an ADR in `docs/decisions/`, and appends rows to both registries. |

### Validators

| Validator | Doctrine rule | Severity |
|---|---|---|
| `no-vendor-types-in-domain` | Operating Rule 2: vendor types do not appear in domain code. Blocks `import AVFoundation`, `import Speech`, etc. in `Services/` and `Views/`. | error |
| `boundary-needs-adr` | Every new row in `BOUNDARIES.md` must have a paired ADR under `docs/decisions/`. | error |
| `swift6-audio-tap-concurrency` | The audio-tap concurrency rule. Catches `installTap` closures that capture non-`Sendable` state, or fail to hop back via `Task { @MainActor in ... }`. | error |
| `observation-binding` | The `@Observable` binding rule. Catches inline `Binding(get:set:)` around `@Observable` properties (registers tracking on the wrong view). | error |

### Templates

| File | Used by |
|---|---|
| `templates/adr-template.md` | `/new-boundary` (and `/plan`) when scaffolding a new ADR. |

## Install

### Via mirdan

```bash
mirdan install boundary-first
```

### Vendor into the iOS repo

```bash
cd /path/to/iphone-app-2
cp -R /path/to/swissarmyhammer/integrations/boundary-first/skills/*       .sah/skills/
cp -R /path/to/swissarmyhammer/integrations/boundary-first/validators/*   .sah/validators/
cp /path/to/swissarmyhammer/integrations/boundary-first/templates/adr-template.md docs/decisions/_template.md
```

## Wiring `avp`

```bash
cd /path/to/iphone-app-2
avp init
avp list
```

`avp` will register the four validators as PreToolUse hooks on `Edit` /
`Write` against Swift files matching the patterns each validator declares.

## How `/new-boundary` works

The user says: "we need to wrap CoreBluetooth for the AirPods battery widget".

The skill walks the checklist:

1. **Scan `BOUNDARIES.md`** for an existing row. If found, stop and report.
2. **Scan `CAPABILITIES.md`** for an existing protocol that fits the
   capability. If found, suggest reusing it.
3. **Else scaffold**:
   - `Audio/Ports/<CapabilityName>.swift` — protocol with one or two
     idiomatic methods.
   - `Audio/Adapters/<Vendor><CapabilityName>.swift` — stub adapter.
   - `Tests/<CapabilityName>Tests.swift` — protocol contract tests via a
     stub adapter.
   - `docs/decisions/000N-<slug>.md` — populated from the ADR template.
   - Append a row to `BOUNDARIES.md` and another to `CAPABILITIES.md`.
4. **Pause for review.** Don't write the feature code that depends on the
   new boundary — the user picks up from the scaffolded files.

The full checklist is in `skills/new-boundary/SKILL.md`. The skill never
writes vendor-import code in domain layers — that's exactly what
`no-vendor-types-in-domain` would block anyway.

## Why this exists

Two reasons:

1. **The checklist in `CLAUDE.md` is doctrine, not muscle memory.** Every
   contributor (human or agent) has to re-derive the steps. A skill makes the
   doctrine executable so it gets followed even at 11pm with a vendor SDK
   crash to debug.

2. **The hard rules ("vendor types don't appear in domain code") only catch
   violations at code-review time.** A validator catches them at edit time,
   when the cost to redirect is one prompt instead of one PR cycle.

## License

MIT OR Apache-2.0
