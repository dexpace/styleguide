---
name: ruby-styleguide
description: Use when writing, editing, or reviewing Ruby (.rb) source in a dexpace project — enforces the dexpace Ruby styleguide (Sorbet # typed: strict, Data.define value objects, frozen by default, 25-line method cap). Also use before committing Ruby, or when asked to review Ruby against the styleguide.
---

# Ruby styleguide

Extends the [Airbnb](https://github.com/airbnb/ruby), [Shopify](https://ruby-style-guide.shopify.dev/), and [rubystyle.guide](https://rubystyle.guide/) chain; where they conflict, the higher authority wins, except the deviations below. Ruby 4.0+; additions: mandatory Sorbet `# typed: strict` with runtime `sig` checking, `find`/`select`/`reduce`/`map` naming.

## When this applies

Editing `*.rb`, or reviewing Ruby. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data and functions, not objects: model state as immutable `Data.define`/`T::Struct` value objects; group behaviour in modules of functions; reserve a stateful `class` for lifecycle resources; mix in a module instead of inheriting for reuse.
2. Explicit over implicit: no `method_missing`, no `define_method` magic, no monkey-patching; every dependency is a parameter; a `sig` makes the type contract explicit; `Hash#fetch` makes a missing key explicit; pass only what differs through keyword arguments.
3. Immutable by default: `frozen_string_literal: true` everywhere, `freeze` every constant, `Data.define` over mutable `Struct`, frozen collections in public signatures; build new values by transformation, never by mutation.
4. Errors are values, handled explicitly: typed `StandardError` subclasses per domain, raised with class and message, chained through `cause`; never rescue `Exception`, never empty/`rescue nil`/modifier rescue; `rescue => error` then handle.
5. Composition over inheritance: `<` is for Liskov-clean hierarchies and `StandardError` trees only; closed polymorphism is a `T::Enum` or `case/in`; reuse is a mixed-in module.
6. Transform, don't mutate: build pipelines with `map`/`select`/`reduce`; reach for `each` only for effects or early exit; take input, return new output; avoid mutating arguments.
7. Always say why: comments and YARD explain reasoning, not mechanics; a `sig` already states the types, so prose never restates them.
8. Assert aggressively: a runtime-checked `sig` asserts every argument and return; add explicit pre/postconditions with `raise` on top — two per method on average; split compound assertions; assert positive and negative space.
9. Limits on everything: 25-line methods, three nesting levels, four params; bound every loop, queue, retry, pool, cache, fan-out; timeouts mandatory on external I/O; no unbounded recursion in library code.
10. Small functions, breathing room: aim 5–15 lines, one level of abstraction each; guard clauses first so the happy path stays flush left; separate logical sections with blank lines.
11. Performance from the outset: work with the runtime's grain — stable object shapes, YJIT, frozen strings, lazy enumerators; batch over per-row work; optimize the slowest resource first (network > disk > memory > CPU).
12. Zero technical debt: do it right the first time — debt never gets paid.

## Language hard rules

- `# typed: strict` on every file; a `sig` on every method (precondition and postcondition in one). `T.let`/`T.cast` carry a why-comment; `T.must`/`T.unsafe` banned outside declared bridges.
- `&.` only where `nil` is a legitimate documented value; `Hash#fetch`/`Array#fetch` over `[]`; never leak `nil` across a boundary; parse don't validate — one parse-constructor takes raw input, everything downstream takes the typed value object.
- Frozen by default: `freeze` every mutable constant, `attr_reader` only on value objects (never `attr_writer`/`attr_accessor`), no `$globals`, no `@@class` variables.
- Model closed sets with `T::Enum`, closed hierarchies with `sealed!`/`abstract!`; close every `case` with `T.absurd`; make illegal states unrepresentable.
- Errors: subclass `StandardError`; never `rescue Exception`; no flow-control by exception; implicit `begin`; never `return` from `ensure`; raise class + message as two args; chain `cause` on rethrow; name the rescue variable `error`.
- Methods: 25-line hard cap (aim 5–15), keyword args over positional, no positional defaults, guard clauses first, `next` over nested block conditionals, `public_send` over `send`, pure by default.
- Idioms: `&:sym`, `tap`/`then`, `case/in`, never `for`, `{}` single-line and `do..end` multi-line, no monkey-patching, `<<~` heredocs, shorthand symbol hash keys, `Time` over `DateTime`, plain literal arrays over `%w`.
- Concurrency: match the model to the workload (Ractors for CPU, Fibers + `async` for I/O, `Mutex` for shared state); bound every pool and queue; never `Timeout.timeout` — use per-call deadlines; share immutable `Data` across boundaries.
- `rubocop` + `rubocop-airbnb` is law (2-space, 100-col, double quotes, trailing commas); `srb tc` is the first suite; Minitest is the second.

## Before you finish — verify

```bash
srb tc                       # type-check gate; must pass clean
rubocop                      # must pass clean (rubocop -a to autocorrect)
bundle exec rake test        # Minitest; must be green
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/ruby/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/ruby/01-formatting-and-tooling.md)
- [02 Naming Conventions](https://github.com/dexpace/styleguide/blob/main/ruby/02-naming-conventions.md)
- [03 Type Safety & Nil Discipline](https://github.com/dexpace/styleguide/blob/main/ruby/03-type-safety-and-nil-discipline.md)
- [04 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/ruby/04-variables-and-declarations.md)
- [05 Methods](https://github.com/dexpace/styleguide/blob/main/ruby/05-methods.md)
- [06 Classes & Data Modeling](https://github.com/dexpace/styleguide/blob/main/ruby/06-classes-and-data-modeling.md)
- [07 Ruby Idioms](https://github.com/dexpace/styleguide/blob/main/ruby/07-ruby-idioms.md)
- [08 Error Handling](https://github.com/dexpace/styleguide/blob/main/ruby/08-error-handling.md)
- [09 Concurrency](https://github.com/dexpace/styleguide/blob/main/ruby/09-concurrency.md)
- [10 API Design](https://github.com/dexpace/styleguide/blob/main/ruby/10-api-design.md)
- [11 Testing](https://github.com/dexpace/styleguide/blob/main/ruby/11-testing.md)
- [12 Module Organization](https://github.com/dexpace/styleguide/blob/main/ruby/12-module-organization.md)
- [13 Resource Management](https://github.com/dexpace/styleguide/blob/main/ruby/13-resource-management.md)
- [14 Documentation](https://github.com/dexpace/styleguide/blob/main/ruby/14-documentation.md)
- [15 Performance](https://github.com/dexpace/styleguide/blob/main/ruby/15-performance.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
