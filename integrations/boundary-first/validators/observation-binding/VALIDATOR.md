---
name: observation-binding
description: Enforce the `@Observable` binding rule. Inline `Binding(get:set:)` closures around `@Observable` properties register tracking against the wrong view because the read happens inside the child's body. Use `@Bindable` for two-way binding to a stored property; otherwise pass `let` values plus explicit callbacks.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "**/Views/**/*.swift"
    - "VoiceMemosClone/Views/**/*.swift"
    - "**/*View.swift"
    - "**/*Sheet.swift"
tags:
  - boundary-first
  - swiftui
  - observation
severity: error
timeout: 30
---

# Observation Binding Rule

SwiftUI's `Observation` framework tracks reads inside `body`. An inline
`Binding(get:set:)` closure around an `@Observable` property causes the read
to happen inside the *child* view's body — so SwiftUI registers the tracking
against the child, not the owner. Result: the parent doesn't re-render when
the property changes.

The rule:

- For **direct two-way binding** to a stored property of an `@Observable`,
  use `@Bindable` on the binding-source view.
- For everything else, pass plain `let` values plus explicit callback
  closures.

## What to flag

In a Swift file matched by the patterns above, flag:

1. **Inline `Binding(get:set:)`** constructed in a `body` expression:
   ```swift
   var body: some View {
       SomeChild(value: Binding(
           get: { observableModel.foo },
           set: { observableModel.foo = $0 }
       ))
   }
   ```

2. **Inline `Binding(get:set:)`** constructed in a stored computed property
   that's read from `body` — same problem one indirection away. Heuristic:
   computed `var` returning a `Binding<...>` whose body references a property
   on an `@Observable` type.

3. **`$model.property` syntax on a non-`@Bindable` model** — `$` projects a
   binding, but only `@Bindable` (or the legacy `@ObservedObject` for
   `ObservableObject`) projects correctly under Observation. If the model
   is declared as `let model: MyObservableModel` (plain `let`), `$model.foo`
   doesn't compile in some contexts; if it's `@State` of an `@Observable`,
   the binding is on the wrong scope.

## What NOT to flag

- **`@Bindable var model: MyObservableModel` followed by `$model.foo`**: this
  is the correct pattern. Allow.

- **`Binding(get:set:)` used to derive a binding from non-`@Observable`
  state** (`@State`, `AppStorage`, a passed-in `Binding<T>`). The rule is
  specific to `@Observable`.

- **Tests / previews**: `*Preview*.swift`, `#Preview` blocks.

## Heuristic for "is this an @Observable property?"

A property is treated as `@Observable`-isolated if it's read off:

- A type annotated with `@Observable` (Swift 5.9+).
- A type annotated with `@Observable` via a macro alias (`@ObservableModel`,
  etc., if the project uses one).
- A property of an `@Environment(MyObservableModel.self)` value.

The validator should resolve the type of the receiver expression to decide.
If it can't be resolved, default to "skip" to avoid false positives —
better an occasional miss than a wall of false negatives.

## Why error

This rule cost a day in Step 5b. The bug presents as "the UI doesn't
update", which is the worst kind of SwiftUI bug — it looks like everything
works, but a downstream state change silently drops. Block.

## Remediation message

> Inline `Binding(get:set:)` around an `@Observable` property registers
> SwiftUI tracking on the child view, so the parent doesn't re-render on
> change.
>
> Pick one of:
>
> 1. **For two-way binding to a stored property**, declare the model as
>    `@Bindable` in the binding-source view:
>    ```swift
>    @Bindable var model: MyObservableModel
>    SomeChild(value: $model.foo)
>    ```
>
> 2. **For derived / read-mostly access**, pass plain values and explicit
>    callbacks:
>    ```swift
>    SomeChild(
>        value: model.foo,
>        onCommit: { newValue in model.foo = newValue }
>    )
>    ```
>
> Reference: project rule in `CLAUDE.md` "Observation rule (Step 5b lesson)".
