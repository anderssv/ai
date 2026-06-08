# ai

Shared AI skills for AI coding assistants (Claude Code, etc).

## Kotlin Skills

These skills originated from [the-example](https://github.com/anderssv/the-example) — a reference codebase demonstrating idiomatic Kotlin practices for TDD, dependency injection, and type-safe validation — and have been migrated here for independent installation and maintenance.

The patterns come from years of real-world Kotlin development and are documented with examples in [the-example's doc/](https://github.com/anderssv/the-example/tree/main/doc) directory.

### Install

```bash
npx skills install anderssv/ai/skills
```

### Available Skills

| Skill | Description |
|-------|-------------|
| **[kotlin-tdd](skills/testing/kotlin-tdd/SKILL.md)** 🧪 | Test-Driven Development with fakes, object mothers, and Testing Through The Domain. Uses TestContext for dependency injection, extension functions for test data, and fakes instead of mocks. |
| **[idiomatic-kotlin](skills/practices/idiomatic-kotlin/SKILL.md)** 🟠 | Enforces idiomatic Kotlin over Java-style patterns. Covers scope functions, collections, data/sealed/value classes, extensions, coroutines, and DSL builders. |
| **[kotlin-context-di](skills/practices/kotlin-context-di/SKILL.md)** 🔌 | Manual dependency injection using SystemContext (production) and TestContext (test doubles) patterns. No DI frameworks needed. |
| **[kotlin-sum-types](skills/domain/kotlin-sum-types/SKILL.md)** 🔀 | Parse, don't validate — using sealed classes for type-safe validation and state representation. Model valid/invalid states explicitly. |

### Supporting Guides

The kotlin-tdd skill includes detailed guides:

- [Fakes](skills/testing/kotlin-tdd/fakes.md) — Working implementations that replace real dependencies
- [Test Setup](skills/testing/kotlin-tdd/test-setup.md) — Extension functions for test data with sensible defaults
- [Testing Through The Domain](skills/testing/kotlin-tdd/tttd.md) — Set up test state using domain operations

### Source

These skills were originally developed in [the-example](https://github.com/anderssv/the-example) and migrated here. The patterns are demonstrated in:

- [TDD with Fakes](https://github.com/anderssv/the-example/blob/main/doc/tdd.md)
- [Manual Dependency Injection](https://github.com/anderssv/the-example/blob/main/doc/manual-dependency-injection.md)
- [Sum Types / Parse Don't Validate](https://github.com/anderssv/the-example/blob/main/doc/sum-types.md)
- [TDD Concepts Overview](https://github.com/anderssv/the-example/blob/main/doc/tdd-concepts-overview.md)

Read more on the [blog](https://blog.f12.no).
