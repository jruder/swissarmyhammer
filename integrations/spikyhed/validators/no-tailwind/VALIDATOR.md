---
name: no-tailwind
description: Enforce SpikyHed Hard Rule §2 — plain CSS with custom properties only. Block Tailwind, Bootstrap, Material UI, and other CSS frameworks.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "apps/**/*.{ts,tsx,js,mjs,cjs,svelte,css}"
    - "packages/**/*.{ts,tsx,js,mjs,cjs,svelte,css}"
    - "**/package.json"
    - "**/tailwind.config.{js,ts,cjs,mjs}"
    - "**/postcss.config.{js,ts,cjs,mjs}"
tags:
  - spikyhed-constitution
  - css
severity: error
timeout: 30
---

# No Tailwind / CSS Frameworks

SpikyHed Hard Rule §2: **Plain CSS with custom properties only. No Tailwind.
No CSS frameworks. Use variables from `docs/design-system/variables.css`.**

## What to flag

1. **Package imports / deps**: `tailwindcss`, `bootstrap`, `@mui/*`,
   `@mantine/*`, `chakra-ui`, `bulma`, `daisyui`, `@unocss/*` in any
   `package.json` under `dependencies` or `devDependencies`.
2. **Config files**: presence of `tailwind.config.{js,ts,cjs,mjs}` or
   `postcss.config.{js,ts,cjs,mjs}` referencing Tailwind plugins.
3. **CSS-in-source**: `@tailwind base;`, `@tailwind components;`,
   `@tailwind utilities;`, `@apply` directives in any `.css`, `.svelte`, or
   `.tsx` file.
4. **Class string atomics in Svelte/TSX**: `class="..."` strings containing
   three or more recognizable Tailwind utility tokens
   (`text-{color}-{shade}`, `bg-{color}-{shade}`, `flex`, `grid`,
   `px-{n}`, `py-{n}`, `mt-{n}`, `space-x-{n}`, etc.). Three or more in one
   string is a near-certain Tailwind site — anything less may be coincidence.

## What NOT to flag

- Plain CSS classes that happen to resemble Tailwind tokens but are defined in
  the project's own CSS (`.card`, `.btn-primary`).
- A markdown file that mentions Tailwind by name.
- `docs/design-system/variables.css` and similar canonical custom-property
  declarations.

## Why

Tailwind atomics couple markup to a utility vocabulary that's owned by a third
party. SpikyHed's design system lives in `docs/design-system/variables.css` as
custom properties, so themability is local. Catching this at the agent
boundary prevents the gradual creep of `class="flex items-center gap-2 px-4"`
into Svelte components.

## Remediation message

> SpikyHed Hard Rule §2 forbids Tailwind and other CSS frameworks. Author
> styles as plain CSS in the component's `<style>` block (Svelte) or a paired
> `.module.css`. Reference custom properties from
> `docs/design-system/variables.css` (e.g. `var(--spacing-md)`,
> `var(--color-fg)`).
