## Ruby Code Style

Binding Ruby rules for all dexpace projects; extends [Airbnb's Ruby Style Guide](https://github.com/airbnb/ruby). Target **Ruby 4.0+**, frozen string literals everywhere, Sorbet `# typed: strict` with runtime-checked signatures, `rubocop` + `rubocop-airbnb` for tooling. Prioritize **correctness**, **explicitness**, **simplicity** — never cleverness, never `nil` leaking out of an interface, never metaprogramming for its own sake.

This guide is platform-agnostic. It covers the Ruby language, its object model, and runtime-neutral idioms. Framework concerns (Rails, Sidekiq, gem packaging beyond the basics) layer on top of this guide and never weaken it.

### Authorities

This guide extends and defers to an ordered chain. Where they conflict, the higher authority wins:

1. **[Airbnb Ruby Style Guide](https://github.com/airbnb/ruby)** — canonical. The primary source of truth for whitespace, method/call shape, conditionals, and naming. Where our guidance collides with it, **Airbnb wins**, except for the deliberate deviations recorded in the ledger below. Airbnb's guide predates modern Ruby in places; the ledger is where we say so.
2. **[Shopify Ruby Style Guide](https://ruby-style-guide.shopify.dev/)** — a decade of large-scale Ruby. It **fills the gaps** Airbnb leaves open and supplies the modern taste (shorthand hash keys, double quotes, `Hash#fetch`, Minitest, squiggly heredocs). Defer to it where Airbnb is silent.
3. **[The Ruby Style Guide (rubystyle.guide)](https://rubystyle.guide/)** — Bozhidar Batsov's community guide, the shared foundation both of the above descend from. The base layer; defer to it where the two above are both silent.
4. **[`rubocop`](https://docs.rubocop.org/) + [`rubocop-airbnb`](https://github.com/airbnb/ruby/tree/master/rubocop-airbnb)** — the tooling embodiment of the above. Its cop baseline and formatter decisions are final; formatting is a non-discussion.
5. **This guide's overlay** — the dexpace layer on top: the Tiger Style discipline (assertion density, bounded everything, no unbounded recursion, zero debt), the method size cap, mandatory Sorbet `# typed: strict` with runtime `sig` enforcement, frozen-by-default values, and `Data.define` value objects with parse-don't-validate constructors.

### Values

**Correctness > performance > developer experience.** This is the root README's ordering; this guide refines "developer experience" into **simplicity > expressiveness**. When rules conflict, that order decides.

- **Correctness** comes first because a fast, simple, expressive program that computes the wrong answer is worthless. In Ruby — a dynamic language — correctness is bought, not given: Sorbet's runtime-checked signatures are the first test suite, and chapter 03 is about not letting `nil` or a wrong type cross a boundary unchecked.
- **Performance** comes before simplicity because the right architecture is chosen once, at design time, and is expensive to retrofit. Work with the grain of the runtime: YJIT, object shapes, frozen strings, lazy enumerators. Optimize the slowest resource first: network > disk > memory > CPU.
- **Simplicity** is the simplest approach that accomplishes the goal: no abstraction for its own sake, no metaprogramming, no cleverness. When two correct, fast-enough designs differ, the simpler one wins.
- **Expressiveness** comes last because Ruby's expressive power is a temptation as much as a gift; clarity is worth nothing if the code is wrong, slow, or needlessly clever.

All four are held to one standard of elegant, well-structured code, enforced through the chapters' rules and exemplars rather than aspired to as a rank in the priority order.

---

## Table of Contents

| # | Document | Scope |
|---|----------|-------|
| 01 | [Formatting & Tooling](./01-formatting-and-tooling.md) | `rubocop-airbnb` baseline; Ruby ≥ 4.0 pinned; `frozen_string_literal` mandatory; 2-space, 100-col, double quotes, trailing commas; `srb tc` typecheck gate; metric caps |
| 02 | [Naming Conventions](./02-naming-conventions.md) | `snake_case`/`CamelCase`/`SCREAMING_SNAKE_CASE`, predicate `?`, bang `!` only paired, no `is_`/`get_`, one class per file, effect-verb naming, no magic numbers |
| 03 | [Type Safety & Nil Discipline](./03-type-safety-and-nil-discipline.md) | `# typed: strict`, `sig` on every method, `T.let`/`T.cast` need a why, `T.must` banned outside bridges, `&.`, `fetch` over `[]`, no `nil` across boundaries, parse don't validate |
| 04 | [Variables & Declarations](./04-variables-and-declarations.md) | local scope, no `$globals`, no `@@class` vars, `freeze` constants, `\|\|=` init (not booleans), memoization caveats, `attr_reader`, no redundant `self.` |
| 05 | [Methods](./05-methods.md) | 25-line cap, keyword args over positional/options-hash, guard clauses, one level of abstraction, `next` over nested blocks, `public_send`, pure by default, 2+ assertions |
| 06 | [Classes & Data Modeling](./06-classes-and-data-modeling.md) | data + functions; modules of functions over class-method bags; `Data.define` value objects; composition via mixins; `T::Enum`/sealed; make illegal states unrepresentable |
| 07 | [Ruby Idioms](./07-ruby-idioms.md) | Enumerable pipelines, `&:sym`, `tap`/`then`, pattern matching `case/in`, no `for`, `{}` vs `do..end`, no monkey-patching, squiggly heredocs, `Time` over `DateTime` |
| 08 | [Error Handling](./08-error-handling.md) | `StandardError` subclasses, never rescue `Exception`, no flow-control by exception, implicit `begin`, no empty/`nil`/modifier rescue, raise class + message, `cause` chaining |
| 09 | [Concurrency](./09-concurrency.md) | GVL realities, `Mutex` + immutable sharing, Ractors for parallelism, Fibers + `async`, bounded pools/queues, deadline timeouts over `Timeout.timeout`, documented races |
| 10 | [API Design](./10-api-design.md) | minimal public surface, keyword args, `sig` on every public method, accept duck types / return frozen concretes, parse at boundaries, deprecation + semver, API symmetry |
| 11 | [Testing](./11-testing.md) | Minitest, `test "..."` blocks, AAA paragraphs, descriptive assertions, fakes over mocks, property tests, determinism via injected `Time`, `srb tc` as the first suite |
| 12 | [Module Organization](./12-module-organization.md) | Zeitwerk autoloading, one class/module per file, nested-module namespacing, no load-time side effects, gem layout, `madge`-style circular-dependency ban |
| 13 | [Resource Management](./13-resource-management.md) | block form for every closable, `ensure` cleanup, bounded pools/queues/caches, IO deadlines, no finalizer reliance, deterministic teardown |
| 14 | [Documentation](./14-documentation.md) | YARD on public API, never restate a `sig`, why-comments, file/class header, `TODO(Name):`, no commented-out code, no block comments |
| 15 | [Performance](./15-performance.md) | YJIT, object shapes, allocation hygiene, `<<` over `+`, lazy enumerators, `size` over `count`, `sub`/`tr` over `gsub`, measure first, network > disk > memory > CPU |

---

## Cross-Cutting Concerns

Security, performance, and git practices are covered in the [root-level code style guide](../README.md). The cross-cutting docs are language-agnostic; this guide adapts them to Ruby.

## The 12 Rules in Ruby

The root README defines twelve rules for every dexpace project. Here they are in Ruby's vocabulary.

1. **Data and functions, not objects.** Model state as immutable `Data.define` value objects; group behaviour into modules of functions and small duck-typed interfaces. Reserve a stateful `class` for lifecycle resources — things you open and close. No inheritance for code reuse; mix in a module instead.
2. **Explicit over implicit.** Code says what it does at the call site and does nothing it did not say. No `method_missing`, no `define_method` magic, no monkey-patching. Every dependency is a parameter, visible in the signature. A `sig` makes the type contract explicit; `Hash#fetch` makes a missing key explicit. Library options follow their documented defaults; callers pass only what differs, through keyword arguments.
3. **Immutable by default.** `frozen_string_literal: true` in every file, `freeze` on every constant, `Data.define` over mutable `Struct`, frozen collections in public signatures. Build a new value by transformation, never by mutation. Mutability is the choice you have to type — that's the right way around.
4. **Errors are values, handled explicitly.** Typed `StandardError` subclasses per domain, raised with a class and message, chained through `cause` on rethrow. Never rescue `Exception`; never an empty rescue, a `rescue nil`, or a modifier rescue that swallows. `rescue => error` then handle — no silent discards.
5. **Composition over inheritance.** `<` is for narrow, Liskov-clean hierarchies and `StandardError` trees, nothing else. Closed polymorphism is a `T::Enum` or a `case/in`; code reuse is a mixed-in module. Small interfaces composed together, never a deep class tree.
6. **Transform, don't mutate.** Build pipelines from `map`/`select`/`reduce`; reach for `each` only for effects or early exit. Methods take input and return new output. Avoid mutating arguments. State changes are explicit, localized, and named.
7. **Always say why.** Comments and YARD explain reasoning, not mechanics; a `sig` already states the types, so prose never restates them. Enforcement notes in the guides name their cop. If you can't say why a line exists, question whether it should.
8. **Assert aggressively.** A runtime-checked `sig` is an assertion on every argument and return. Add explicit preconditions and postconditions with `raise` on top: minimum two per method on average. Split compound assertions; assert positive and negative space; pair-assert a property two independent ways.
9. **Limits on everything.** Methods cap at 25 lines (cop-enforced), nesting at three levels, params at four. Bound every loop, queue, retry, pool, cache, and fan-out. Timeouts are mandatory on external I/O. No unbounded recursion in library code — all execution provably bounded.
10. **Small functions, breathing room.** Aim for 5–15 lines, one level of abstraction each. Guard clauses first so the happy path stays flush left. Separate logical sections with blank lines — whitespace is free, cramped code is unreadable.
11. **Performance from the outset.** Design-time is when 1000× improvements are cheap. Work with the grain of the runtime — stable object shapes, YJIT, frozen strings, lazy enumerators. Batch over per-row work. Optimize the slowest resource first: network > disk > memory > CPU.
12. **Zero technical debt.** What exists meets the design goals. **Perfection over technical debt — debt never gets paid.** Do it right the first time; the second chance may never come.

## Deviations from Upstream

This guide takes Airbnb's Ruby Style Guide as canonical. The entries below are genuine deviations — places we override Airbnb, mostly because its guide predates modern Ruby — plus additions Airbnb does not address. Each is recorded so it can be revisited surgically.

| Rule | Upstream position | Our position | Why |
|---|---|---|---|
| Hash key syntax | Airbnb: hash-rocket symbol keys, `{ :one => 1 }` | Shorthand symbol keys, `{ one: 1 }`; hash-rockets only when a key is not a symbol | Airbnb predates ubiquitous 1.9 syntax; shorthand is the modern community and Shopify default. See [07-ruby-idioms.md](./07-ruby-idioms.md). |
| String quotes | Airbnb: unspecified / mixed | Double quotes everywhere | One rule, no decision per literal; matches Shopify. See [01-formatting-and-tooling.md](./01-formatting-and-tooling.md). |
| Word arrays | Airbnb: "use `%w` freely" | Prefer plain literal arrays of double-quoted strings | Explicit and grep-able; `%w` is a second string syntax to learn. See [07-ruby-idioms.md](./07-ruby-idioms.md). |
| Collection method names | Airbnb: prefer `detect` over `find`, `reduce` over `inject` | `find`, `select`, `reduce`, `map` | The community/Shopify canonical set; `detect` was an ActiveRecord-disambiguation that no longer earns its keep. See [07-ruby-idioms.md](./07-ruby-idioms.md). |
| Class methods | Airbnb: `def self.method` | Grouped in a single `class << self` block | Keeps `private` working for class methods and collects them in one place; matches Shopify. See [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md). |
| Static typing | Not addressed | Mandatory Sorbet `# typed: strict` with runtime `sig` checking | Owner decision; maximum safety in a dynamic language — static *and* runtime enforcement. See [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md). |
| Method size | No upstream cap | 25-line hard cap, cop-enforced; aim 5–15 | Owner decision; Tiger Style discipline, the Ruby-scaled sibling of Go's 70 and Kotlin's 60. See [05-methods.md](./05-methods.md). |
| Test framework | Airbnb: unspecified | Minitest | One framework, the Shopify and core-Ruby default. See [11-testing.md](./11-testing.md). |

## Influences

- **[Airbnb Ruby Style Guide](https://github.com/airbnb/ruby)** — canonical authority. Where our guidance collides, Airbnb wins (save the deviations above).
- **[Shopify Ruby Style Guide](https://ruby-style-guide.shopify.dev/)** — large-scale modern taste; fills the gaps Airbnb leaves open.
- **[The Ruby Style Guide (rubystyle.guide)](https://rubystyle.guide/)** — the community foundation both descend from.
- **[Sorbet](https://sorbet.org/)** — the type system; runtime-checked signatures as the first test suite.
- **[RuboCop](https://docs.rubocop.org/)** — the single lint and format baseline for this guide.
- **TigerBeetle Tiger Style** — assertion density, the method size cap, limits on everything, no unbounded recursion, zero technical debt.

## Applying Style Changes

When adopting a new rule or migrating away from a deprecated pattern, apply the change at the **file / module level or larger** — never mix two styles within the same file. A half-migrated file is more confusing than either end state.
