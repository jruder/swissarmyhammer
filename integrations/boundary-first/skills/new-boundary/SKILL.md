---
name: new-boundary
description: Onboard a new external boundary (vendor SDK, OS API, network service, ML model) under the Boundary-First Architecture doctrine. Use when the user says "wrap X", "add a port for X", "we need to integrate X", "new boundary for X", or anything implying a new external surface. Scaffolds protocol + adapter + tests + ADR + registry rows BEFORE any feature code lands.
license: MIT OR Apache-2.0
compatibility: Requires the `code_context` MCP tool for symbol search, the `shell` MCP tool for file scaffolding, and a project layout matching the Boundary-First doctrine (`Audio/Ports/`, `Audio/Adapters/`, `BOUNDARIES.md`, `CAPABILITIES.md`, `docs/decisions/`). Designed for Swift / iOS but the layout is project-shaped, not language-shaped.
metadata:
  author: boundary-first
  version: "1.0.0"
---

# New Boundary

The Operating Rule of Boundary-First Architecture is non-negotiable:

> Every external boundary is wrapped in a protocol/interface we own.
> Vendor types do not appear in domain-level code.
> New features start by identifying their boundaries and writing the protocols.

This skill performs the "Before writing new code" checklist from the doctrine
and leaves the user with: a port, an adapter stub, contract tests, an ADR,
and two updated registry files. No domain or view code is written until the
boundary is settled.

## When to use

The user wants to introduce a new external surface. Trigger phrases include:

- "wrap X" / "add a port for X"
- "we need to integrate X"
- "the new feature needs X" where X is an SDK, OS framework, network endpoint,
  ML model, file format, or hardware capability
- "new boundary for X"

Do NOT use this skill for changes that stay inside an already-wrapped
boundary — those are normal feature work.

## Protocol

### 1. Identify the boundary

Ask the user (if not already clear):

- **What capability** does it provide? (one-line domain-language description)
- **Vendor / source**: Apple SDK? Open source? Internal API?
- **Swap risk**: low (mature API), medium, high (brand-new, churn likely).

### 2. Scan `BOUNDARIES.md`

```
code_context search symbol <BoundaryName>
```
plus grep `BOUNDARIES.md` for the vendor name and the capability description.

If a matching row exists:

- If it's marked `✅ wrapped`, point the user at the existing port; STOP.
- If it's marked `🔜 wrap in Step N`, this is the right time — proceed.
- If it's marked `❌ never integrated` or `❌ direct, accept until Step N`,
  ask the user whether they're now ready to wrap (graduating from `❌` to `✅`).

### 3. Scan `CAPABILITIES.md`

Look for a protocol that already provides the same capability under a
different vendor. If one exists, the new boundary is a NEW adapter for an
EXISTING port, not a new port — STOP the scaffold and tell the user.

### 4. Pick names

- **Protocol**: a domain noun ending in a role suffix (`Denoiser`, `Transcriber`,
  `Recorder`, `SyncBackend`, `AuthGate`). Avoid vendor names in the protocol.
- **Adapter**: `<VendorOrTechnology><ProtocolName>` (e.g. `RNNoiseDenoiser`,
  `SpeechAnalyzerTranscriber`, `CloudKitSyncBackend`).
- **ADR slug**: kebab-case domain phrase, NOT vendor (e.g. `denoiser-choice`,
  not `rnnoise-integration`).

### 5. Scaffold the files

In order:

a. **Protocol** at `Audio/Ports/<ProtocolName>.swift` (or the project's
   ports dir):
   ```swift
   import Foundation

   protocol <ProtocolName>: Sendable {
       func <verb>(_ input: <DomainType>) async throws -> <DomainType>
   }
   ```
   Use ONLY Foundation types and domain types in the protocol signature.
   Never `AVAudioPCMBuffer`, never `MLModel`, never `URLSession.DataTask`.
   If the natural signature needs a vendor type, wrap it in a domain
   `struct` first.

b. **Adapter** at `Audio/Adapters/<AdapterName>.swift`:
   ```swift
   import Foundation
   import <VendorFramework>

   final class <AdapterName>: <ProtocolName> {
       func <verb>(_ input: <DomainType>) async throws -> <DomainType> {
           // TODO: implement using <VendorFramework>
       }
   }
   ```

c. **Contract test** at `Tests/<ProtocolName>Tests.swift`:
   ```swift
   import Testing
   @testable import VoiceMemosClone

   struct <ProtocolName>Contract {
       let sut: <ProtocolName>

       @Test func roundTripsDomainInput() async throws {
           // TODO: contract assertion every adapter must satisfy
       }
   }
   ```
   The contract test runs against every adapter. New adapters MUST extend
   it, not write their own ad-hoc tests.

d. **ADR** at `docs/decisions/000N-<slug>.md` (use the next free `000N`):
   Populate from `templates/adr-template.md` — context, decision,
   alternatives, swap risk, last reviewed date.

### 6. Update registries

Append a row to `BOUNDARIES.md`:

```
| <N+1> | <Boundary> | <Vendor> | <Capability> | <Risk> | 🔜 wrapping now | <Alternatives> | <today's date> |
```

Append a row to `CAPABILITIES.md`:

```
| `<ProtocolName>` | `Audio/Ports/<ProtocolName>.swift` | <Capability description> | <Inputs> | <Outputs> | `<AdapterName>` | <consumers> |
```

Also add a one-paragraph semantic description at the bottom under
`### <ProtocolName>` — this is what `/plan` reads when matching future feature
requests against existing capabilities.

### 7. STOP

Tell the user: "Scaffolded port + adapter + contract test + ADR + registry
rows. Adapter body is stubbed. Pick up from `Audio/Adapters/<AdapterName>.swift`
to implement, or hand off to `/implement` once the next feature task is
on the kanban."

DO NOT write the feature code that depends on the new boundary in the same
turn. The boundary settles first.

## Hard refusals

- **No vendor imports in `Services/` or `Views/`.** If the user asks to "just
  use AVFoundation directly here", refuse and offer the port.
- **No "temporary" direct usage.** Operating Rule 4: implementations are
  interchangeable behind the protocol. If it's worth doing, it's worth
  wrapping. The validator `no-vendor-types-in-domain` will block it anyway.
- **No ADR-less boundary.** If the user wants to skip the ADR, refuse —
  `boundary-needs-adr` will block the PR otherwise. Offer to write a short
  ADR (3-paragraph minimum) if they want it lighter.

## Example

User says: "we need to wrap CoreBluetooth for the AirPods battery widget"

Actions:

1. Identify: capability = "read accessory battery level"; vendor = Apple
   CoreBluetooth; swap risk = low.
2. Scan `BOUNDARIES.md` → no row for CoreBluetooth. ✓
3. Scan `CAPABILITIES.md` → no battery-reading protocol exists. ✓
4. Pick names:
   - Protocol: `AccessoryBatteryReader`
   - Adapter: `CoreBluetoothBatteryReader`
   - ADR slug: `accessory-battery-source`
5. Scaffold:
   - `Audio/Ports/AccessoryBatteryReader.swift` with a `read() async throws
     -> AccessoryBattery` method
   - `Audio/Adapters/CoreBluetoothBatteryReader.swift` (stub)
   - `Tests/AccessoryBatteryReaderTests.swift` (contract)
   - `docs/decisions/0030-accessory-battery-source.md` (filled from template)
6. Append row to `BOUNDARIES.md` (#30, CoreBluetooth, …, 🔜 wrapping now, …,
   2026-05-18). Append row to `CAPABILITIES.md`. Add semantic paragraph.
7. Tell user: "Boundary scaffolded as #30; pick up at
   `CoreBluetoothBatteryReader.swift` to implement the read path."

## Troubleshooting

### Two protocols seem to fit the same capability

- **Cause**: someone wrapped the same capability twice, OR the capability is
  too broad and should be split.
- **Solution**: STOP. Treat this as a refactor, not a new boundary. Open an
  ADR proposing the merge/split before scaffolding.

### The protocol signature would naturally take a vendor type

- **Cause**: the domain hasn't yet been modeled — you're tempted to "just pass
  the AVAudioPCMBuffer through" because there's no domain `AudioFrame`.
- **Solution**: define the domain type first. Add it to
  `VoiceMemosClone/Models/` or `Audio/Models/`. THEN write the protocol
  against the domain type. This is how the boundary stays at the boundary.
