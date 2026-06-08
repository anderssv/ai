---
name: idiomatic-kotlin
description: Enforces idiomatic Kotlin over Java-style patterns when writing or reviewing Kotlin code. Use when generating, reviewing, or refactoring Kotlin. Covers scope functions, collections, data/sealed/value classes, extensions, coroutines, and DSL builders.
domain: kotlin
tags: [kotlin, idiomatic, style, best-practices]
---

STARTER_CHARACTER = 🟠

Apply these principles whenever writing or reviewing Kotlin code. Produce Kotlin that uses the language's features rather than translating Java patterns into Kotlin syntax.

## Core Principles

- **Nullable types over Optional.** Kotlin's `?` integrates with safe calls, elvis, and smart casts. `Optional` is for Java interop boundaries only.
- **Immutability by default.** `val` over `var`. `List` over `MutableList`. `data class copy()` over mutation. Expression-form `if`/`when`/`try` to avoid `var`.
- **The type system prevents bugs.** Sealed classes make illegal states unrepresentable. Value classes prevent parameter swapping. Exhaustive `when` (no `else` on sealed hierarchies) catches missing cases at compile time.
- **Declare intent, not mechanics.** Functional collection operations (`map`, `filter`, `associateBy`, `groupBy`, `flatMap`) over imperative loops. `buildString` over `StringBuilder`. `use()` over try-finally. `require`/`check` over manual throws.
- **Properties are the interface.** No `getX()`/`setX()` methods. Custom accessors for validation. Backing property pattern (`_items`/`items`) for mutable-to-immutable exposure.
- **Structured concurrency.** `suspend` functions over callbacks. Scoped coroutines over `GlobalScope`. `launch` for fire-and-forget, `async` only when you `await`. `Flow` over callback-based streams. `runBlocking` only at top-level entry points.

## Scope Functions

Pick based on what you reference (`it` vs `this`) and what you return (lambda result vs context object):

- `let` -- transform nullable or introduce scoped variable. Returns lambda result. Ref: `it`.
- `run` -- compute a result using object members. Returns lambda result. Ref: `this`.
- `apply` -- configure an object (set properties). Returns the object. Ref: `this`.
- `also` -- side effects (logging, validation). Returns the object. Ref: `it`.
- `with` -- group calls on a non-null object. Returns lambda result. Ref: `this`.

Anti-patterns to catch:
- Nesting scope functions (pyramid of `let`/`run` blocks) -- flatten with safe call chains instead
- `apply` for side effects (use `also`)
- `let` when `this`-based access is cleaner (use `run` or `apply`)
- `with` on nullable receivers (use `?.run` instead)
- `?.let` for a single method call (simple `if` with smart cast is often clearer)

## Collections

- `filter`/`map` over manual loops that accumulate into mutable lists
- `associateBy`/`associate`/`groupBy`/`associateWith` over manual map building
- `flatMap` over nested loops
- `asSequence()` for large collections with chained operations or short-circuiting (`take`, `first`)
- Prefer `for` loop over `forEach` for simple iteration (per official conventions), except in call chains or on nullable receivers
- Return `List`, not `MutableList` from public APIs

## When NOT to Use Data Classes

Data classes derive `equals`/`hashCode` from ALL constructor properties. This is wrong for entities with identity (database rows, domain objects with IDs where equality is by ID alone). Use data classes for value objects.

## Sealed Classes and Interfaces

- Prefer sealed classes/interfaces over enums when subclasses carry different data
- Each subclass carries exactly the data it needs -- illegal states are unrepresentable
- Never use `else` on exhaustive `when` over sealed hierarchies -- compiler catches missing cases at compile time
- Use `sealed interface` when subclasses need independent class hierarchies

## Value Classes

- Use `@JvmInline value class` to prevent parameter swapping (e.g. `CustomerId` vs `ProductId`)
- Value classes provide compile-time type safety with zero runtime overhead
- `typealias` gives no type safety -- use it only to simplify complex generic types (e.g. `typealias EventHandler<T> = (T) -> Unit`)

## Extension Functions

- Prefer extension functions over utility/helper objects (`StringUtils.isPalindrome` → `String.isPalindrome()`)
- Extensions should be conceptually "methods the type should have" -- restrict visibility to where needed
- Avoid extensions on `Any`, or extensions that are unrelated to the receiver's domain
- Core behavior → member function (participates in overriding). Cross-cutting/client-specific → extension (statically resolved)

## Coroutines

- `suspend` functions over callbacks
- Scoped coroutines (`coroutineScope { }`) over `GlobalScope` -- avoids orphaned coroutines ignoring cancellation
- `launch` for fire-and-forget; `async` only when you `await` the result
- `runBlocking` only at top-level entry points (`main()`, tests, framework bridges)
- `Flow` over callback-based streams -- integrates with cancellation and backpressure

## String Handling

- String templates (`"Hello, ${user.name}"`) over concatenation
- `buildString { }` over `StringBuilder`
- Kotlin string interpolation for logging -- SLF4J `{}` placeholders are a Java concern (avoid cost of string concatenation when log level disabled); in Kotlin, string templates are cheap and more readable
- Multiline strings with `""".trimIndent()` over escaped `\n`

## Companion Objects

Empty companion objects on data classes are intentional when test extension functions target them:

```kotlin
// Production domain class
data class Poll(
    val id: UUID,
    val title: String,
) {
    companion object  // Empty -- enables test extension functions
}

// Test scope: extension function on Companion
fun Poll.Companion.valid() = Poll(
    id = UUID.randomUUID(),
    title = "Test Poll",
)

// Usage in tests
val poll = Poll.valid().copy(title = "Custom")
```

This pattern enables test data builders via `ClassName.valid()` with `.copy()` for variations. Don't flag empty companion objects as anti-patterns -- they're the hook for test extension functions.

## Default Parameter Values

Default parameter values are an anti-pattern. They obscure call-site intent and make it harder to trace which value is actually used. They should only be used in very rare cases. A default value may serve as an intermediate step during a refactoring (e.g., adding a new parameter without updating all call sites at once), but it must always be removed before committing -- all call sites should pass explicit values. In test code, defaults are acceptable for object mothers and test data builders, where they reduce noise and focus each test on what varies.

## Official Style Conventions

These are from the Kotlin coding conventions that are easy to miss:

- Use named arguments for multiple same-type parameters and boolean parameters
- Use trailing commas at declaration sites
- Omit `Unit` return type, semicolons, redundant `public`
- Use `0..<n` over `0..n-1` for open-ended ranges
- Single-expression functions: use `= expr` form, omit braces and `return`
- For library code: always specify visibility and return types explicitly

## Anti-Pattern Reference

`references/anti-patterns.md` contains a comprehensive catalogue of before/after examples for all topics in this skill (nullability, scope functions, collections, data classes, sealed classes, value classes, extension functions, properties, coroutines, string handling, object declarations, DSL builders, default parameters). Load it when reviewing code or when you need concrete examples to back up a recommendation.

## Related Skills

Load these skills when working on Kotlin -- they complement idiomatic-kotlin:

- **kotlin-tdd** -- Kotlin-specific TDD patterns, test infrastructure, and test context setup
- **tdd** -- General test-driven development process
- **kotlin-context-di** -- Manual dependency injection using context patterns for Kotlin
- **kotlin-sum-types** -- Parse-don't-validate with sealed classes
- **hexagonal-architecture** -- Ports & adapters architecture for structuring domain, application, and adapter layers
