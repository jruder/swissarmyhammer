---
name: python-deps-declared
description: Enforce SpikyHed Hard Rule §18 — every Python import must resolve to a declared dep (requirements.txt or pyproject.toml), the standard library, or the package itself. Origin: 2026-04-30 kg-resolver crash.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "tools/**/*.py"
    - "apps/**/*.py"
    - "**/requirements*.txt"
    - "**/pyproject.toml"
tags:
  - spikyhed-constitution
  - python
  - incident-2026-04-30
severity: error
timeout: 60
---

# Python Dependencies Must Be Declared

SpikyHed Hard Rule §18: **Every `import X` / `from X import Y` in any Python
file under `tools/*/` or `apps/*/` must resolve to a declared dep
(`requirements.txt` or `pyproject.toml [project.dependencies]` /
`[project.optional-dependencies]`), to the standard library, or to the
package itself.**

Origin: 2026-04-30 kg-resolver shipped with `requests` missing from
`requirements.txt`; image built clean, deployed, then crashed in prod with
`ModuleNotFoundError`. ~4 hours of build + deploy time burned.

## What to flag

When the agent edits a `.py` file OR a `requirements*.txt` / `pyproject.toml`:

1. **Find the package root.** Walk up from the file until you hit a
   `requirements*.txt`, `pyproject.toml`, or `.python-no-deps`.

2. **If `.python-no-deps` is present**, skip — this directory is an exempt
   host-Python helper (rare).

3. **Collect imports** in the file:
   ```
   import <name>
   import <name> as <alias>
   import <name>.<submodule>
   from <name> import <symbol>
   from <name>.<submodule> import <symbol>
   ```
   Reduce to the top-level package name (`<name>`).

4. **Collect declared deps** from `requirements*.txt` (PEP 508 lines) and
   `pyproject.toml` (`[project.dependencies]` and
   `[project.optional-dependencies]`).

5. **Classify each import**:
   - Standard library — OK. Use a current stdlib list (Python 3.12+).
   - Same-package import (`<package_root_name>` matches one of the imports'
     top-level) — OK.
   - Declared dep — OK.
   - **Anything else** — flag.

6. **If the agent is editing `requirements.txt` / `pyproject.toml`** to
   REMOVE a dep, re-run the check across all `.py` files in the package root
   to make sure no live import would be orphaned.

## What NOT to flag

- Files with a top-of-file `# python-no-deps` magic comment (rare; same
  semantics as the marker file but file-scoped).
- Files in `.venv/`, `venv/`, `__pycache__/`, `build/`, `dist/`.
- Test files inside a package whose deps are declared in
  `[project.optional-dependencies.test]` — those count as declared.

## Stdlib reference

A maintained list is bundled with this validator. Currently keyed to Python
3.12 stdlib. Notable common-but-not-stdlib packages that get flagged: `requests`,
`yaml` (use `pyyaml`), `pydantic`, `numpy`, `httpx`, `aiohttp`, `pandas`,
`sqlalchemy`, `psycopg`, `redis`, `boto3`, `cryptography`.

## Why error

The kg-resolver incident proved that "the image builds" is not enough. A
declared-deps check is the cheapest possible runtime-crash prevention — it
runs in seconds and catches the bug at the edit site, before CI, before build,
before deploy. Block.

## Remediation message

> Python import `<name>` in `<file>` doesn't resolve to a declared dep,
> stdlib, or this package. Origin of this rule: 2026-04-30 kg-resolver
> shipped without `requests` declared, crashed in prod.
>
> Add the dep to the right manifest:
>
> - `requirements.txt`: append `<name>==<pinned-version>`.
> - `pyproject.toml`: add `<name>` to `[project.dependencies]` or
>   `[project.optional-dependencies.<group>]`.
>
> If this Python file is a host-Python-only helper (rare), add a
> `.python-no-deps` marker file in the directory with a one-line reason.
