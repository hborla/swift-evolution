# Control default actor isolation inference

* Proposal: [SE-NNNN](NNNN-control-default-actor-isolation.md)
* Authors: [Holly Borla](https://github.com/hborla), [John McCall](https://github.com/rjmccall)
* Review Manager: TBD
* Status: **Awaiting review**
* Vision: [[Prospective Vision] Improving the approachability of data-race safety](https://forums.swift.org/t/prospective-vision-improving-the-approachability-of-data-race-safety/76183)
* Implementation: On `main` under `-enable-experimental-feature UnspecifiedMeansMainActorIsolated`
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

This proposal introduces a new compiler setting for inferring `@MainActor`
isolation by default within the module.
## Motivation

A lot of code is effectively “single-threaded”. For example, most executables, such as apps, command-line tools, and scripts, start running on the main actor and just stay there unless some part of the code actually does something concurrent (like creating a `Task`). If there isn’t any use of concurrency, the entire program will run sequentially, and there’s no risk of data races — every concurrency diagnostic is necessarily a false positive! It would be good to be able to take advantage of that in the language, both to avoid annoying programmers with unnecessary diagnostics and to reinforce progressive disclosure. Many people get into Swift by writing these kinds of programs, and if we can avoid needing to teach them about concurrency straight away, we’ll make the language much more approachable.

The easiest and best way to model single-threaded code is with a global actor. Everything on a global actor runs sequentially, and code that isn’t isolated to that actor can’t access the data that is. All programs start running on the global actor `MainActor`, and if everything in the program is isolated to the main actor, there shouldn’t be any concurrency errors.
## Proposed solution

We believe that the right solution to these problems is to allow code to opt in to being “single-threaded” by default, on a module-by-module basis. This would change the default isolation rule for unannotated code in the module: rather than being non-isolated, and therefore having to deal with the presumption of concurrency, the code would instead be implicitly isolated to `@MainActor`. Code imported from other modules would be unaffected by the current module’s choice of default. When the programmer really wants concurrency, they can request it explicitly by marking a function or type as `nonisolated` (which can used on any declaration as of [SE-0449](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0449-nonisolated-for-global-actor-cutoff.md)), or they can define it in a module that doesn’t default to main-actor isolation. This doesn’t fundamentally change anything about Swift’s isolation model; it just flips the default, effectively creating a model in which code is single-threaded except where it explicitly requests concurrency. Modules that don’t want this could of course continue to use the current rules.

Making a module be isolated to the main actor by default would directly fix several of the false positive problems listed above for single-threaded code. Global variables would default to being isolated to the main actor, avoiding the diagnostic when they’re declared. Functions in the module would also default to being isolated to the main actor, allowing them to freely use both those isolated global variables and any main-actor-isolated functions and variables imported from the platform SDK. Class overrides and protocol conformances aren’t quite so easy, but we think we can extend them in ways that allow a natural solution with main actor isolation. We’ll get to how later in this document.



## Detailed design

### `enable` statements to control default actor isolation per file

A new `enable` statement can be used at the top-level in a file to set the default actor isolation for all declarations in the file:

```swift
// FileA.swift
enable default-isolation(MainActor)

struct S {}
```


```swift
// FileB.swift

enable default-isolation(nonisolated)
```

### `-default-isolation` compiler flag for setting default actor isolation per module

The `-default-isolation` flag can be used to control the default actor isolation for all files in the module. This flag effectively infers a `enabe default-isolation` statement in every file in the module with the argument provided to the flag. The only valid arguments to `-default-isolation` are `MainActor` and `nonisolated`. It is an error to specify both `-default-isolation MainActor` and `-default-isolation nonisolated`. If no `-default-isolation` flag is specified, the default isolation for the module is `nonisolated`.
### Inferred `@MainActor` isolation


Not applied to
* Declarations with explicit or inferred isolation
* Anything inside an actor
* Declarations that cannot have global actor isolation, e.g. typealiases, import statements, enum cases...

### Opting out of `@MainActor`


```swift
nonisolated struct S {
  // everything within 'S' is 'nonisolated'
}
```

## Source compatibility

Changing the default actor isolation for a given module or source file is a source incompatible change. The default isolation will remain the same for existing projects unless they explicitly opt into `@MainActor` inference by default via `-default-isolation MainActor` or `enable default-isolation(MainActor)`.

## ABI compatibility

This proposal has no ABI impact on code that does not adopt `-default isolation` or `enable default-isolation`.
## Implications on adoption

This proposal does not change the adoption implications of adding `@MainActor` to a declaration that was previously `nonisolated` and vice versa. The source and ABI compatibility implications of changing actor isolation are documented in the Swift migration guide's [Library Evolution](https://github.com/apple/swift-migration-guide/blob/29d6e889e3bd43c42fe38a5c3f612141c7cefdf7/Guide.docc/LibraryEvolution.md#main-actor-annotations) article.
## Future directions

### Expand `enable` statements to other build settings

The `enable` statement can be expanded to other build settings that make sense to enable on a per-file bases, such as compiler flags that enable additive language features or control diagnostic behavior:

* `enable strict-concurrency(complete)`
* `enable upcoming-feature(ExistentialAny)`
* `enable experimental-feature(UnspecifiedMeansMainActor)`
* `enable Wwarning(deprecation)`

Compiler flags can be named as they currently are, with flags prefixed with `-enable` allowed to drop the redundant prefix.

### Allow `-default-isolation` to specify a custom global actor

The `-default-isolation` flag could allow a custom global actor as the argument.

## Alternatives considered

### Only infer `@MainActor` on global and static variables

