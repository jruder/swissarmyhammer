---
name: swift6-audio-tap-concurrency
description: Enforce the Swift 6 audio-tap concurrency rule. AVAudioEngine tap closures run on the real-time audio thread. Under Swift 6 strict concurrency, they must be installed from a `nonisolated static func`, capture only `Sendable` types, and hop UI writes via `Task { @MainActor in ... }`.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "**/*.swift"
tags:
  - boundary-first
  - swift6
  - concurrency
severity: error
timeout: 60
---

# Swift 6 Audio Tap Concurrency

The audio tap closure runs on AVAudioEngine's real-time render thread. Under
Swift 6's strict concurrency model, that thread is a separate isolation
domain. Closures captured by `installTap` MUST:

1. Be installed from a `nonisolated static func` (or a `nonisolated`
   instance method on a `Sendable` actor-or-struct).
2. Capture ONLY `Sendable` values. No `self` capture on a non-`Sendable`
   class, no `@MainActor`-isolated locals.
3. Hop any UI state mutation back via `Task { @MainActor in ... }`.

The pattern lives in `Services/RecorderService.swift` →
`installTap(on:format:bufferSize:processor:)`.

## What to flag

Inspect each Swift file for `installTap(...)` calls:

```swift
node.installTap(onBus: bus, bufferSize: size, format: format) { buffer, time in
    /* closure body */
}
```

Flag if ANY of the following are true:

1. **The enclosing function is NOT `nonisolated static func`** (or
   `nonisolated` on a `Sendable` type). The enclosing function's name should
   be `installTap(on:format:bufferSize:processor:)` per the project's
   helper pattern.

2. **The closure captures `self`** explicitly (`[self]`) or implicitly
   (`self.someProperty` inside the body), AND the enclosing type is not
   marked `Sendable`.

3. **The closure captures any `@MainActor`-bound property** read directly.
   Heuristic: a captured identifier whose declaration is annotated
   `@MainActor` or lives on an `@Observable` `@MainActor`-isolated class.

4. **The closure mutates UI state without a `Task { @MainActor in ... }`
   hop**. Heuristic: assignment to a property whose declared type is one of
   the `@Observable` view-model classes in the project.

## What NOT to flag

- Tap closures inside `*Tests.swift` — tests legitimately wire engines
  synchronously without strict concurrency concerns.

- Closures whose body is a single call to a `Sendable` method on a
  `Sendable` processor type (e.g. `processor.process(buffer)`). That's the
  pattern this rule is meant to encourage.

- `installTap` calls where the closure is a stored property of `nonisolated`
  type — those are the equivalent of the static helper.

## Why error

This rule exists because Step 4 burned a real day. Swift 6 strict
concurrency is a compile-time gate; a violation either fails to compile or,
worse, ships a subtle data race past the actor boundary. The
"nonisolated static func" pattern is the *only* one the project sanctioned.

## Remediation message

> Audio-tap closure violates the Swift 6 concurrency rule. Tap closures run
> on the real-time render thread, a distinct isolation domain.
>
> Move the `installTap` call into a `nonisolated static func` helper:
>
> ```swift
> nonisolated static func installTap(
>     on node: AVAudioNode,
>     format: AVAudioFormat,
>     bufferSize: AVAudioFrameCount,
>     processor: any SendableTapProcessor
> ) {
>     node.installTap(onBus: 0, bufferSize: bufferSize, format: format) { buffer, time in
>         processor.process(buffer)
>         Task { @MainActor in
>             // UI state hop here, e.g. update an @Observable
>         }
>     }
> }
> ```
>
> Captured `processor` must be `Sendable`. Reference implementation:
> `VoiceMemosClone/Services/RecorderService.swift` (search for the existing
> `installTap` helper).
