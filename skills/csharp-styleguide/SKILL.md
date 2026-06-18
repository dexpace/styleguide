---
name: csharp-styleguide
description: Use when writing, editing, or reviewing C# (.cs) source in a dexpace project — enforces the dexpace C# styleguide (NRT as law, records + pattern matching, async all the way, 70-line method cap). Also use before committing C#, or when asked to review C# against the styleguide.
---

# C# styleguide

Extends the [.NET Runtime C# Coding Style](https://github.com/dotnet/runtime/blob/main/docs/coding-guidelines/coding-style.md); where they conflict, the runtime wins, except the recorded deviations: no `I` prefix on first-party interfaces, no `Async` suffix, a stricter null-forgiving ban, and a 70-line method cap. Target C# 14 / .NET 10.

## When this applies

Editing `*.cs`, or reviewing C#. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data and functions, not objects: `record` types and `readonly struct`s for state, `static` classes of pure methods and small interfaces for behavior. Stateful `class` only for open/close lifecycle resources. No inheritance for reuse.
2. Explicit over implicit: every dependency a constructor parameter, every annotation honest. No `dynamic`, no reflection magic in domain code, no `!` papering over null. Library options follow documented defaults.
3. Immutable by default: `readonly` fields, `init`-only properties, `record` over mutable `class`, `IReadOnlyList<T>` in public signatures. Update with `with`, never by mutation.
4. Errors are values, handled explicitly: specific exception types, cause chained as `InnerException`, context on rethrow. Catch only what you handle, never bare `Exception` without a `when` filter. Opt-in `Result<T, TError>` where failure is routine. No empty `catch`; a discarded `Task` is a build error.
5. Composition over inheritance: `abstract`/`virtual` only for a closed hierarchy backing a discriminated union. Reuse is delegation through an injected interface. Small interfaces composed, never a deep class tree.
6. Transform, don't mutate: LINQ pipelines (`Select`/`Where`/`Aggregate`); `foreach` only for effects or early exit, never in a measured hot path. Input in, new output out. State changes explicit, localized, named.
7. Always say why: XML doc and `//` comments explain reasoning, not mechanics; `<example>` on non-obvious public API. If you can't say why a line exists, question it.
8. Assert aggressively: `Debug.Assert` for invariants, the `ArgumentNullException.ThrowIfNull` / `ArgumentOutOfRangeException.ThrowIf*` family for public preconditions. ≥2 per method on average; split compound checks; assert positive and negative space.
9. Limits on everything: 70-line method cap (analyzer-enforced), nesting two levels, three at most. Bound every loop, queue, retry, pool, cache, fan-out. Timeouts mandatory on external I/O via a `CancellationToken`. No recursion in library code.
10. Small functions, breathing room: aim 10–30 lines, one level of abstraction. Guard clauses first so the happy path stays flush left. Blank lines between logical sections.
11. Performance from the outset: work with the grain of the JIT — stable types, few allocations, `Span<T>` over copies, `ValueTask` where it pays. Optimize the slowest resource first: network > disk > memory > CPU.
12. Zero technical debt: do it right the first time. Perfection over technical debt — debt never gets paid.

## Language hard rules

- NRT is law: `<Nullable>enable</Nullable>` solution-wide, `<TreatWarningsAsErrors>true`. Null-forgiving `!` banned outside declared bridges and `[MemberNotNull]`-style proofs. No `#nullable disable`. `default!` only at a proven boundary. `dynamic` banned; accept `object` only at the edge and pattern-match inward.
- Data + functions: immutable `record` DTOs, `required` + `init` for construction completeness; choose `record`, `record struct`, or `class` by value semantics. `sealed` by default. No inheritance for reuse.
- Errors: specific exception types, never bare `catch (Exception)` without a `when` filter; `throw;` to rethrow (never `throw ex;`) with inner-exception chaining; `ArgumentNullException.ThrowIf*` guards; opt-in `Result` for expected failures; no control-flow exceptions; `OperationCanceledException` is cooperative cancellation, not an error.
- Concurrency: async all the way down; `async void` banned (return `Task`/`Task<T>`/`ValueTask`); `CancellationToken` last parameter plus timeout everywhere; `ConfigureAwait(false)` in library code; never `.Result`/`.Wait()`/`GetAwaiter().GetResult()`; bounded `Channel<T>`, bounded fan-out.
- Naming: PascalCase types and members, `_camelCase` instance fields (`s_`/`t_` statics), camelCase locals and parameters; no `I` prefix on first-party interfaces (BCL interfaces keep theirs); no `Async` suffix; language keywords over BCL types; `nameof` over string literals.
- Idioms: `switch` expressions over if/else chains, LINQ pipelines, collection expressions `[..]` with spread, ranges `..` and indices `^`, `is not null`, `??`/`??=`/`?.`, raw strings, `nameof`.
- Method size: **70-line hard cap** (analyzer-enforced), aim 10–30 at one level of abstraction; options record at three or more parameters; no boolean behavior flags.
- Formatting: `dotnet format` + `.editorconfig` are law — Allman braces, four spaces, file-scoped namespaces; `<AnalysisLevel>latest-Recommended</AnalysisLevel>`.

## Before you finish — verify

```bash
dotnet format --verify-no-changes   # formatting is non-negotiable
dotnet build -warnaserror           # NRT + analyzers as errors
dotnet test
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/csharp/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/csharp/01-formatting-and-tooling.md)
- [02 Naming Conventions](https://github.com/dexpace/styleguide/blob/main/csharp/02-naming-conventions.md)
- [03 Nullability & the Type System](https://github.com/dexpace/styleguide/blob/main/csharp/03-nullability-and-the-type-system.md)
- [04 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/csharp/04-variables-and-declarations.md)
- [05 Methods & Functions](https://github.com/dexpace/styleguide/blob/main/csharp/05-methods-and-functions.md)
- [06 Types & Data Modeling](https://github.com/dexpace/styleguide/blob/main/csharp/06-types-and-data-modeling.md)
- [07 C# Idioms](https://github.com/dexpace/styleguide/blob/main/csharp/07-csharp-idioms.md)
- [08 Error Handling](https://github.com/dexpace/styleguide/blob/main/csharp/08-error-handling.md)
- [09 Concurrency & Async](https://github.com/dexpace/styleguide/blob/main/csharp/09-concurrency.md)
- [10 API Design](https://github.com/dexpace/styleguide/blob/main/csharp/10-api-design.md)
- [11 Testing](https://github.com/dexpace/styleguide/blob/main/csharp/11-testing.md)
- [12 Project & Assembly Organization](https://github.com/dexpace/styleguide/blob/main/csharp/12-project-organization.md)
- [13 Resource Management](https://github.com/dexpace/styleguide/blob/main/csharp/13-resource-management.md)
- [14 Documentation](https://github.com/dexpace/styleguide/blob/main/csharp/14-documentation.md)
- [15 Performance](https://github.com/dexpace/styleguide/blob/main/csharp/15-performance.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
