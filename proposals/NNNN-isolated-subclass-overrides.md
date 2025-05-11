# Isolated subclass overrides

* Proposal: [SE-NNNN](NNNN-isolated-subclass-overrides.md)
* Authors: [Holly Borla](https://github.com/hborla), [Author 2](https://github.com/swiftdev)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Vision: [Improving the approachability of data-race safety](/visions/approachable-concurrency.md)
* Implementation: [swiftlang/swift#NNNNN](https://github.com/swiftlang/swift/pull/NNNNN) or [swiftlang/swift-evolution-staging#NNNNN](https://github.com/swiftlang/swift-evolution-staging/pull/NNNNN)
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

This proposal enables subclasses to add actor isolation when overriding nonisolated superclass methods.

## Motivation

TODO

## Proposed solution

TODO

## Detailed design

### Statically isolated overrides

* A global-actor-isolated subclasses can override nonisolated methods from the superclass with global-actor isolation.
* All isolated overrides must share the isolation of the subclass.
* The class must have isolated initializers.
* The class must not conform to `Sendable`.

The class is always part of the global actor's region, and conversions to the superclass type will result in a superclass value that is also part of the global actor's region. This ensures that isolated overrides cannot be called from outside the actor.

### `@preconcurrency` subclasses

A `@preconcurrency` class inheritance is scoped to the implementation of the overridden methods in the subclass. A `@preconcurrency` superclass can be written at the primary declaration of a subclass, and override checker diagnostics about actor isolation will be suppressed:

```swift
import XCTest

@MainActor
final class MainActorType {
  init() {}
}

@MainActor
final class MainActorTestCase: @preconcurrency XCTestCase {
  private var type: MainActorType?

  // Main actor-isolated instance method 'setUp()' has different actor isolation from nonisolated overridden declaration
  @MainActor
  override func setUp() {
    super.setUp()

    type = MainActorType()
  }
}
```

Like other `@preconcurrency` annotations, if no diagnotsics are suppressed, a warning will be emitted at the `@preconcurrency` annotation stating that the annotation has no effect and it should be removed.

## Source compatibility

TODO

## ABI compatibility

TODO

## Implications on adoption

TODO

## Alternatives considered

TODO

## Acknowledgments

TODO
