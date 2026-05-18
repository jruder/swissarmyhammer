---
name: no-vendor-types-in-domain
description: Enforce Boundary-First Operating Rule 2 — vendor types do not appear in domain-level code. Block imports of vendor frameworks (AVFoundation, Speech, CoreML, CoreBluetooth, etc.) inside `Services/` and `Views/`.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "**/Services/**/*.swift"
    - "**/Views/**/*.swift"
    - "VoiceMemosClone/Services/**/*.swift"
    - "VoiceMemosClone/Views/**/*.swift"
    - "VoiceMemosClone/CIM/**/*.swift"
    - "VoiceMemosClone/Calendar/**/*.swift"
    - "VoiceMemosClone/Notifications/**/*.swift"
    - "VoiceMemosClone/HIL/**/*.swift"
tags:
  - boundary-first
  - architecture
severity: error
timeout: 30
---

# No Vendor Types in Domain Code

Boundary-First Operating Rule 2: **Vendor types do not appear in domain-level
code.** The domain (`Services/`, `Views/`, and similar) talks only to ports
in `Audio/Ports/`. Adapters in `Audio/Adapters/` are the only place vendor
SDKs are touched.

## What to flag

In any file matched by the patterns above, flag:

1. **Top-level vendor imports**:
   ```swift
   import AVFoundation
   import AVFAudio
   import Speech
   import CoreML
   import Vision
   import CoreBluetooth
   import CoreLocation
   import CoreMotion
   import HealthKit
   import EventKit
   import Contacts
   import LocalAuthentication
   import CloudKit
   import ActivityKit
   import WidgetKit
   import DeviceActivity
   import FamilyControls
   import Cinematic
   import WatchConnectivity
   import StoreKit
   import Network            // raw sockets
   import CryptoKit          // exception below
   ```
   Plus any third-party module imports (anything that isn't `Foundation`,
   `SwiftUI`, `Combine`, `Observation`, `SwiftData`, `OSLog`, `Testing`,
   `XCTest`, or a project-local module).

2. **Type references in signatures** without an import (could be
   compile-error code under review):
   - Function/method parameters, return types, or property types whose
     unqualified name matches a known vendor type (`AVAudioPCMBuffer`,
     `MLModel`, `SFSpeechRecognizer`, `SpeechAnalyzer`, `EKEvent`,
     `EKEventStore`, `CMPedometerData`, `HKQuantitySample`, `CLLocation`,
     `CNContact`, etc.).

3. **Vendor type aliases**: `typealias FooBar = AVAudioPCMBuffer` defined in
   a domain file. Type aliases don't make vendor types domain types.

## What NOT to flag

- **`Foundation`, `SwiftUI`, `Combine`, `Observation`, `SwiftData`, `OSLog`,
  `Testing`, `XCTest`** — these are platform plumbing, not vendor capability
  SDKs. The doctrine treats them as part of the language surface.

- **Project-local modules** — the imports listed in `BOUNDARIES.md` under
  "wrapped" status are still imported in `Audio/Adapters/`, but in domain
  layers we only import the project's own port modules.

- **Tests**: `*Tests.swift`, `*Spec.swift`, anything under a `Tests/`
  directory. Tests legitimately wire adapters together to exercise the full
  stack, including the vendor side.

- **`AppIntents/*.swift`**: intent definitions are themselves a vendor
  surface (`AppIntents` framework). They count as adapters per the doctrine
  (`each intent is a thin adapter over domain`) and are excluded.

- **Specific exempt files** (rare, must be explicit): files whose first
  non-empty line is `// boundary-exempt: <one-line reason>`. The exemption
  comment is the contract — removing it re-enables the check.

## Why error, not warning

This is the bedrock rule of the architecture. A warning would erode the
boundary one "I'll clean it up later" at a time. Block.

## Remediation message

> Boundary-First Operating Rule 2: vendor types don't appear in domain code.
> File `<file>` is in the domain layer (`Services/` / `Views/` / …) but
> imports `<vendor framework>`.
>
> Fix path:
>
> 1. Identify which capability the vendor type provides (e.g. AVAudioEngine
>    → audio capture).
> 2. Check `CAPABILITIES.md` for an existing port that fits.
> 3. If one exists, depend on the port instead — pass the adapter in via the
>    initializer. If none exists, run `/new-boundary` to scaffold one.
>
> Domain code asks for capabilities, not vendor types.
