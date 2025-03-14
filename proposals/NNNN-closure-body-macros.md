# Closure Body Macros

* Proposal: [SE-NNNN](NNNN-closure-body-macros.md)
* Authors: [Holly Borla](https://github.com/hborla)
* Review Manager: TBD
* Status: **Awaiting review**
* Implementation: [swiftlang/swift#79980](https://github.com/swiftlang/swift/pull/79980), [swiftlang/swift-syntax#3016](https://github.com/swiftlang/swift-syntax/pull/3016)
* Previous Proposal: [SE-0415: Function Body Macros][SE-0415]
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

This proposal extends function body macros to closures to allow macros to augment closure bodies with more functionality.

## Motivation

Many of the same use cases for function body macros also apply to closures, including:

* Augmenting function bodies to perform logging/tracing, check preconditions, or establish invariants.
* Replacing function bodies with a new implementation based on the one provided. For example, moving the body into a closure that is executed somewhere else, or treating the body as written as a domain specific language that the macro "lowers" to executable code.

For example, consider the `@AssumeMainActor` macro example from [SE-0415: Function Body Macros][SE-0415]:

```swift
@attached(body) macro AssumeMainActor() = #externalMacro(...)

extension MyView: SomeDelegate {
  @AssumeMainActor
  nonisolated func onSomethingHappened(event: Event) {
    myView.title = newTitle(processing: event)
  }
}

// expands to

extension MyView: SomeDelegate {
  nonisolated func onSomethingHappened(event: Event) {
    MainActor.assumeIsolated {
      myView.title = newTitle(processing: event)
    }
  }
}
```

This `@AssumeMainActor` macro would also be applicable to closures that are not statically isolated to `@MainActor` but are guaranteed to always be called on the main actor at runtime.

## Proposed solution

I propose allowing body macros to apply to closures. For example, the `@AssumeMainActor` body macro can be written on a closure in the closure signature:

```swift
acceptClosure { @AssumeMainActor in
  // closure body
}
```

The macro is given the closure expression syntax as input, and produces a list of statements that replace the closure body. For example, the `@AssumeMainActor` would replace the closure body to place all of the statements inside a call to `MainActor.assumeIsolated`:

```swift
acceptClosure {
  MainActor.assumeIsolated {
    // closure body
  }
}
```

## Detailed design

### Declaring closure body  macros

Closure body macros are declared in the same way that function body macros are, using the `body` role:

```swift
@attached(body) macro AssumeMainActor() = #externalMacro(...)
```

### Implementing closure body macros

This proposal adds the following requirement to the `BodyMacro` protocol:

```swift
  /// Expand a macro described by the given custom attribute and
  /// attached to the given closure and evaluated within a
  /// particular expansion context.
  ///
  /// The macro expansion can replace the body of the given closure.
  static func expansion(
    of node: AttributeSyntax,
    providingBodyFor closure: ClosureExprSyntax,
    in context: some MacroExpansionContext
  ) throws -> [CodeBlockItemSyntax]
```

The given closure body will be replaced by the code items produced from the macro implementation.

To preserve source compatibility of existing `BodyMacro` conformances, the requirement has a default implementation that throws an error, which will result in a compiler error if a programmer tries to apply the macro to a closure. The default implementation is deprecated to prompt macro authors to provide an implementation of the requirement.

### Closure body macro application

Like other closure attributes, body macro attributes are written in the closure signature before the parameter list:

```swift
{ @MyMacro in
  ...
}
```

#### Type checking closure body macro attributes

Function body macros on closures are different from other attached macros in local scope because they must be expanded before type checking, and they can be part of the same expression that introduces values used in the macro argument list.

This poses a challenge for type checking closure body macro attributes; you may want to use values from the surrounding context as macro arguments, but the macro attribute must be expanded before those values are type checked. For example:

```swift
f(0) { z in
  { @Traced(z) (x, y) in
    x + y
  }
}
```

In the above example the `@Traced` body macro must expand the closure body before type checking the expression, and the type of `z` is not known until the the overload for `f` is fully resolved. Waiting to resolve `@Traced` until an overload for `f` is selected means that the `@Traced` macro would have the potential to be expanded multiple times for each potential type of `z`. Resolving `@Traced` during overload resolution also does not guarantee that the type of `z` will be known when resolving the macro, because both single and multi-statement closures support inferring closure parameter types from the closure body. For example:

```swift
struct G<T> {}

func acceptClosure<T>(_: (G<T>) -> Void) {}

func useInt(_: G<Int>) {}

func test() {
  acceptClosure { x in
    print("hello")
    useInt(x) // the type of 'T' is inferred as 'Int' here
  }
}
```

To solve these problems, closure body macros delay type checking attribute arguments. When resolving the macro attribute, all non-literal argument values will be made opaque, and concrete types are only used on the opaque argument values if they are explicitly written with `as`. Macro resolution must be able to disambiguate macro overloads from the following aspects of the macro arguments:

* Literal argument values
* Explicit argument types with written with `as`
* Explicit generic arguments
* Argument labels or the size of the argument list

Resolving a macro attribute is allowed to have types that cannot be inferred due to opaque argument values, but macro resolution must be able to disambiguate macro overloads.

For example, the following macro attribute is ambiguous:

```swift
@attached(body) macro MyMacro(_: Int) = #externalMacro(...)
@attached(body) macro MyMacro(_: String) = #externalMacro(...)

func acceptClosure(_: (Int) -> Void) {}

func applyMacro() {
  acceptClosure { x in
    { @MyMacro(x) in // error
      ...
    }()
  }
}
```

Type checking a closure body macro proceeds as follows:

1. Before type checking an expression, all closure body macros are resolved and expanded. This step is recursive, because body macro expansions may contain closures with other body macros attached.
2. After the closure that the body macro is attached to has a resolved type (which may include not-yet-resolved types in structural positions), the macro arguments are type checked.
3. After macro arguments are type checked, the expanded macro body is type checked.

This approach preserves the property that body macros are expanded only once without sacrificing too much expressivity in macro attributes.

## Source compatibility

This is an additive change that has no impact on source compatibility.

## ABI compatibility

This is an additive change with no impact on ABI.

## Implications on adoption

This feature can be freely adopted and un-adopted in source code with no deployment constraints and without affecting source or ABI compatibility.

## Future directions

### Concurrency macros

This proposal could be used to provide a set of macros for eliminating concurrency boilerplate, including wrapping entire function bodies in an unstructured task, a task group, `assumeIsolated`, etc. For example, a `@Task` macro could be used to wrap an entire function or closure body in a new task:

```swift
Button(label: ... ) { @Task in
  let image = await downloadImage()
  ...
}

// expands to

Button(label: ...) {
  Task {
    let image = await downloadImage()
    ...
  }
}
```

## Alternatives considered

### Restricting closure body macro arguments to literals

A straightforward solution to the macro argument type checking problem is to simply restrict macro arguments to literal values, because type checking literals cannot be impacted by the surrounding local context. However, this approach is too restrictive for the uses cases of function body macros on closures. For example, an `@Task` macro might need the ability to specify an actor value to enqueue the task on in the case where the programmer does not want the default actor isolation inference from context, or the ability to specify a task executor preference:

```swift
acceptClosure { @Task(on: MainActor.self) in 
  ...
}

acceptClosure { @Task(executorPreference: myExecutor) in
  ...
}
```

### Type check macro arguments outside of the expression context

Another solution to the macro argument type checking problem is to type check the macro arguments as if they were written outside of the expression. Instead, the macro would be type checked within the context of the innermost enclosing declaration. This is an attractive option due to its simplicity, but it may cause confusion if variables are shadowed within the expression, because a variable reference in the macro attribute that is copied to the expansion might resolve to difference variables in the attribute versus the expansion. For example:

```swift
func acceptClosure(_: (String) -> Void) {}

func shadow(a: Int) {
  acceptClosure { a in
    { @MyMacro(a) in
      ...
    }()
  }
}
```

When resolving `@MyMacro(a)`, `a` would have type `Int`, but if `a` is used in the macro expansion, it would have type `String`.

### Expanding closure body macros during overload resolution

Another approach to the macro argument type checking problem is delaying resolving a closure body macro until the structure of the closure is known during overload resolution. This approach has the following tradeoffs:

1. Closure body macros could be expanded multiple times, which would lead to poor overload resolution performance.
2. This approach does not guarantee that all of the argument types of the closure will be known at the point of resolving the macro, because closure parameter types can be inferred from the first use within the closure body, including nested closures.

## Acknowledgments

Thank you to Pavel Yaskevich for helping debug issues in the implementation and brainstorming strategies to the argument type checking problem.

[SE-0415]: /proposals/0415-function-body-macros.md