# ADR <NNNN>: <Decision title in domain language, not vendor name>

- **Status**: Proposed | Accepted | Superseded by ADR-NNNN | Deprecated
- **Date**: YYYY-MM-DD
- **Authors**: <name(s)>

## Context

What's the situation that forces a decision? What are the constraints? What's
the *capability* the project needs (described in domain terms, not vendor
terms)?

## Decision

What did we decide? Be specific:

- **Protocol**: `<ProtocolName>` at `Audio/Ports/<ProtocolName>.swift`
- **Adapter chosen now**: `<AdapterName>` at `Audio/Adapters/<AdapterName>.swift`
- **Alternative adapters allowed**: `<AdapterName2>`, `<AdapterName3>`

## Alternatives Considered

For each option, one paragraph: what it is, why it could fit, why we didn't
pick it. Don't strawman — give each alternative a fair shake. Include the
**no-op / fallback** option (e.g. `NoOpDenoiser`) even if it's obviously
inferior; calling it out makes the decision tree explicit.

| Option | Pros | Cons | Why not (now) |
|---|---|---|---|
| <AdapterName> | … | … | (Chosen) |
| <Alternative 1> | … | … | … |
| <Alternative 2> | … | … | … |
| No-op / fallback | predictable, zero deps | doesn't actually do the thing | Reserved for tests / disabled-feature path |

## Swap Risk

How likely is it that we'll need to replace the chosen adapter within 12
months?

- **Low**: mature API, stable vendor, narrow surface.
- **Medium**: new API or unfamiliar territory; expect minor churn.
- **High**: brand-new, sparse docs, vendor tooling broken, or open ecosystem
  with active alternatives.

What would force a swap? (concrete triggers — version bump, performance
budget breach, vendor sunset)

## Consequences

- **For domain code**: still talks to `<ProtocolName>` — unchanged.
- **For tests**: contract tests in `Tests/<ProtocolName>Tests.swift` run
  against every adapter, including the no-op.
- **For build size / dependencies**: <vendor SDK adds N MB, requires
  iOS X.Y+, etc.>
- **For permissions / privacy**: <Info.plist entries, App Privacy Report
  rows>.
- **For incident response**: if `<AdapterName>` breaks in prod, the swap is
  `<other adapter>` — implementer should be able to ship the swap inside one
  PR.

## Review Cadence

- **Last reviewed**: YYYY-MM-DD
- **Next review trigger**: quarterly, OR on vendor SDK major version, OR on
  any incident touching this boundary.

## References

- `BOUNDARIES.md` row #<N>
- `CAPABILITIES.md` entry for `<ProtocolName>`
- Vendor docs: <url>
- Related ADRs: <list>
