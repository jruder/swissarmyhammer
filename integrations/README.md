# SwissArmyHammer Integration Packages

Reference packages that show how to wrap a project's house rules and operating
doctrine as SwissArmyHammer skills and `avp` validators. Each subdirectory is a
ready-to-publish `mirdan` package — drop it into the registry with
`mirdan publish <subdir>`, or copy its contents into a project's `.sah/` folder
for a local install.

| Package | What it does | Target project |
|---|---|---|
| [`spikyhed/`](./spikyhed/) | Encodes the SpikyHed Constitution + Hard Rules + Jackalope guard as `avp` validators, and packages the deploy / nightly-auto-ship / Drizzle-blast-radius flows as SAH skills. | TypeScript monorepos modeled after `jruder/spikyhed_final`. |
| [`boundary-first/`](./boundary-first/) | Packages the Boundary-First Architecture doctrine as a `/new-boundary` skill plus four `avp` validators (no-vendor-types-in-domain, boundary-needs-adr, Swift 6 audio-tap concurrency, `@Observable` binding). | Swift / iOS projects that follow the boundary-first pattern. |

## Why this exists

A project's `CLAUDE.md` or `AGENTS.md` is essentially a long prose list of
operating rules. SAH lets that prose become executable:

- **Rules that say "never do X"** → an `avp` validator that blocks edits doing X.
- **Rules that say "to do Y, follow these steps"** → a SAH `SKILL.md` an agent
  can invoke by name.
- **Rules that span teams** → a `mirdan` package every contributor installs once.

The two packages here are concrete examples of that transform, derived from two
real codebases with mature agent doctrine.

## Layout

Each package is a directory containing:

```
<package>/
  README.md            # Description + install instructions
  skills/<name>/SKILL.md
  validators/<name>/VALIDATOR.md
  validators/<name>/rules/*.md    # Optional fine-grained rules
  templates/*.md       # Optional templates the skills reference
```

## Publishing

```bash
cd integrations/spikyhed
mirdan publish .
```

## Installing into a downstream project

```bash
# From the registry (after publish):
mirdan install spikyhed-constitution
mirdan install boundary-first

# Or vendor it directly:
cp -R integrations/spikyhed/skills/* /path/to/repo/.sah/skills/
cp -R integrations/spikyhed/validators/* /path/to/repo/.sah/validators/
```
