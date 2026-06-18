# Ruby styleguide — full audit checklist

One heading per chapter. Walk every bullet against the code under review.

### 01 — Formatting & Tooling

- Pin Ruby ≥ 4.0 via `.ruby-version`; commit `Gemfile.lock`; CI runs `bundle install --frozen`.
- `rubocop` + `rubocop-airbnb` is the baseline; cop overrides carry a chapter/rule comment; `rubocop --autocorrect-all` is the only sanctioned style fix.
- Wire Sorbet: `sorbet-static` + `sorbet-runtime`; `srb tc` is a CI gate.
- Open every file with `# frozen_string_literal: true` then `# typed:` sigil, blank line before code; no other magic comments.
- 2-space soft tabs, Unix LF, UTF-8, final newline, no trailing whitespace.
- Line length ≤ 100 columns; extract a local before wrapping; lone URLs are the only exception.
- Double-quoted strings always; no `%q`/`%Q`/`%{}` for ordinary strings.
- One expression per line; no semicolons except a single-line empty class body.
- Spaces around operators and after punctuation, inside `{ }`; none inside `[]`/`()` or after unary operators.
- Multiline literals/calls: one element per line, trailing comma, closing delimiter on its own line; chains break with a leading dot.
- Metric caps live in `.rubocop.yml`: MethodLength 25, ParameterLists 4, BlockNesting 3, plus AbcSize and CyclomaticComplexity.

### 02 — Naming Conventions

- `snake_case` for methods, variables, symbols, file names, directories.
- `CamelCase` for classes and modules; keep established acronyms all-caps.
- `SCREAMING_SNAKE_CASE` for constants; carry the unit or scale in the name; no magic numbers.
- One class or module per file; filename is the `snake_case` of the constant path.
- Predicate methods end in `?` and return a real boolean; non-boolean methods never end in `?`.
- Use `!` only when a non-bang counterpart exists; the bang marks the dangerous variant.
- No `is_`/`has_`/`get_` prefixes; use an effect verb (`fetch_`/`load_`/`find_`) for I/O or fallible calls.
- Name side-effecting methods with an effect verb; name pure transforms as nouns or adjectives.
- Name the binary-operator parameter `other`; name `reduce` block args mnemonically.
- Use `_` or `_`-prefixed names for throwaway and unused variables.
- Prefer `allowlist`/`denylist`, `primary`/`replica`; extract magic numbers into named constants.

### 03 — Type Safety & Nil Discipline

- `# typed: strict` on every file; lower sigils are tracked deviations.
- A `sig` on every method, treated as runtime precondition and postcondition.
- Declare every ivar with `T.let` in `initialize`, in a consistent order.
- `T.nilable` is a last resort; raise or return an empty collection rather than `nil`.
- `&.` only where `nil` is a legitimate, documented value.
- `Hash#fetch`/`Array#fetch` for keys and indices that must exist.
- Ban `T.must`/`T.unsafe` outside declared bridges; every bridge use carries a why-comment.
- `T.cast` is the single sanctioned cast and needs a validation above it plus a why-comment.
- Parse, don't validate: one constructor takes raw input, everything downstream takes the typed value object.
- Model closed sets with `T::Enum`; never free symbols or strings for domain states; exhaust with `T.absurd`.
- `T::Struct`/`Data.define` for typed value objects at boundaries; a `Hash` crossing a boundary is unparsed input.
- Type the negative space: a return type is non-nil unless `nil` genuinely models a domain absence.

### 04 — Variables & Declarations

- Keep local-variable scope minimal; declare near first use; extract a method when a name needs a generic placeholder.
- No `$global` variables and no Perl special variables; use named regex captures.
- No `@@class` variables; use a class instance variable or a frozen constant.
- Freeze every mutable constant; `frozen_string_literal` does not cover arrays or hashes.
- `||=` for lazy init, but never on booleans — test `nil?` explicitly for those.
- Memoize only pure, idempotent results; never `return` from inside a memoizing `begin` block.
- Treat assignment-in-condition as a truth test only when parenthesized.
- Use shorthand self-assignment operators (`+=`, `<<`, `||=`, `&&=`).
- `attr_reader` for trivial accessors; never `attr_writer`/`attr_accessor` on value objects.
- Drop redundant `self.` except where the language requires it.
- Declare constants at the top, frozen, typed with `T.let`; magic numbers become named constants.

### 05 — Methods

- 25-line hard cap; aim 5–15; every callable and block body counts.
- One level of abstraction per method — orchestrate named steps or perform primitive work, never both.
- Guard clauses first; keep the happy path flush left.
- Keyword arguments over positional; never a bare positional default; no options hash.
- ≤ 4 parameters; a behaviour-forking boolean is two methods.
- Prefer `next` (and `return`) over nested conditional bodies in loops and blocks.
- No single-line method defs; endless methods only for trivial pure one-liners.
- Omit redundant `return`; reserve explicit `return` for guards.
- Parentheses on `def` when it takes parameters; omit them when it takes none.
- Pure by default; side effects live at the edges; inject time and randomness.
- `public_send` over `send`; bare `send` is a design smell outside test helpers.
- Assert aggressively: 2+ per method on average via the project-wide `Assert.that`/`Assert.fail`.

### 06 — Classes & Data Modeling

- Data and functions, not objects; reserve classes for lifecycle resources you open and close.
- Prefer a module of functions over a class of only class methods; `extend self` for simple utilities.
- `Data.define` for immutable value objects; `T::Struct` when Sorbet must track the type; never a mutable record where either works.
- Write parse-don't-validate constructors that return frozen, valid instances; make `new` the only entry through `parse`.
- Compose via `include`/`prepend`/`extend` and `Forwardable`; subclass only for `StandardError` trees and Sorbet base types.
- Never `@@` class variables; use a class instance variable for class-level state.
- Group all class methods in a single `class << self` block with `extend T::Sig`.
- Make illegal states unrepresentable: `T::Enum`, `sealed!`/`abstract!`, `T.absurd` to exhaust.
- Define and depend on small, role-named duck-typed interfaces as Sorbet `abstract!` modules.
- `attr_reader` only on value objects; never a setter that lets an invariant break after construction.
- Respect Liskov whenever you inherit — extend with `super`, never replace shared logic.
- Keep classes small and single-responsibility; a name needing "and" is two classes.

### 07 — Ruby Idioms

- Build pipelines with `map`/`select`/`reject`/`reduce`/`find`; reach for `each` only for effects or early exit; name stages past three steps.
- Use `&:symbol` when the block is a single no-argument method call.
- `tap` for side-effecting inspection in a chain; `then` to pipe a value through an expression; never `tap` to mutate-and-carry.
- `case/in` for structural decomposition; one-line `=>`/`in` where they read.
- Never `for`; use block iterators.
- `{ ... }` for single-line and chained blocks; `do ... end` for multi-line; never chain `do..end`, never multi-line `{ }`.
- Never monkey-patch core classes; a scoped refinement with a why-comment is the last resort.
- Avoid needless metaprogramming; write `ruby -w`-clean code; explicit beats clever.
- Squiggly heredocs `<<~` for multi-line strings.
- Shorthand symbol hash keys; symbols over strings as keys; `Hash#fetch` with a default for absent keys.
- String interpolation over concatenation; brace `@ivars`/`$globals` in interpolation; never `.to_s` inside interpolation.
- `Time` over `DateTime`; `Time.iso8601` over `Time.parse`; keep timestamps in UTC.
- Plain literal arrays over `%w`/`%i`; `first`/`last` over `[0]`/`[-1]`; `&&`/`||`/`!` over `and`/`or`/`not`.

### 08 — Error Handling

- Build a typed `StandardError` hierarchy rooted at one project base, two levels deep, carrying typed context fields.
- Never rescue `Exception`; rescue `StandardError` or a named subclass.
- No exceptions for flow control; check conditions, return `nil`/booleans for absence and yes/no.
- Use implicit `begin`: method-level `rescue`/`ensure` without an explicit `begin` block.
- No empty rescue, no `rescue nil`, no modifier rescue; a deliberate swallow names the class and the why.
- Raise with a class and a message as two arguments; never a bare `RuntimeError` string in app code.
- Never `return` from an `ensure` block.
- Never `return` from a `begin`/`end` used in an assignment context.
- Name the rescue variable `error`, not `e`.
- Chain `cause` on rethrow; attach context fields; wrap raw third-party exceptions in a domain class.
- Prefer a standard-library exception where one fits; introduce a new class only when callers must distinguish it.
- `ensure` for cleanup; assertions raise `Assert::InvariantViolation`; keep programmer errors out of business-logic rescues.

### 09 — Concurrency

- Know the GVL; match the model to the workload (threads for I/O, Ractors for CPU, Fibers for high-volume I/O).
- Use Ractors for CPU parallelism, bounded to core count; share only frozen objects.
- `Mutex#synchronize` for shared mutable state; prefer immutable sharing and message passing.
- Fibers + `async` for high-concurrency I/O; structured tasks drain automatically.
- Bound every pool and queue with named constants; `FixedThreadPool`/`SizedQueue`/`Async::Semaphore`, never raw `Thread.new` in loops.
- Never `Timeout.timeout`; apply per-call deadlines via `with_timeout` or client timeout options; set connect and read timeouts.
- Cap retries with bounded exponential backoff and jitter; named constants; raise a typed error on exhaustion.
- Use immutable `Data` value objects across thread and Ractor boundaries; annotate the invariant.
- Minimize the critical section; never hold a lock across I/O — fetch then lock.
- Prefer `concurrent-ruby` primitives over hand-rolled synchronization.
- Deterministic teardown: join threads, `shutdown` + `wait_for_termination` pools, `close` queues.

### 10 — API Design

- Minimize the public surface; default to `private`, promote only on genuine external need.
- A `sig` on every public method as the written, runtime-checked contract.
- Keyword arguments on every public method; positionals are order-dependent and break on extension.
- Accept the narrowest duck-typed interface; return a concrete frozen value.
- Parse at the boundary into a value object; never pass a raw `Hash` or scalar deeper.
- Don't return `nil` to mean "absent" where an empty collection or raised error is clearer; document legitimate `T.nilable`.
- Keep API symmetry: paired operations share names, argument shapes, and error contracts.
- Deprecate deliberately: runtime `warn` + `@deprecated` tag; remove only on the next major; version by semver.
- Defaults follow the library's documented defaults; callers pass only what differs.
- Keep argument shapes stable: required keywords first, optional after; never reorder or add a required keyword without a major bump.
- Define value-object protocol methods via `Data.define`/`T::Struct`; don't hand-roll `==`/`hash`/`to_h`.

### 11 — Testing

- Minitest with `test "..." do` blocks; require helpers and subjects explicitly; mirror `lib/` in `test/`.
- AAA structure as blank-line-separated paragraphs; one behaviour per test; behavioural test names.
- Descriptive assertions (`assert_equal`, `assert_predicate`, `refute_nil`) over bare `assert`; `expected` before `actual`.
- One aspect per test case; split compound tests so failures are independent.
- Assert a positive outcome; never `assert_nothing_raised`.
- Prefer fakes over mocks; double only at genuine external boundaries; name doubles for what they are.
- Property-based tests for pure functions and value objects; bound the iteration count and seed.
- Determinism: inject the clock and randomness; never read `Time.now` inside a unit under test.
- `srb tc` is the first test suite; a type error is a test failure; `# typed: strict` on every test file.
- Test the negative space: assert errors at boundaries, check messages and side-effect absence, pair-assert properties.
- Tests are fast and isolated: bound data, no shared mutable state, no order dependence.

### 12 — Module Organization

- Use Zeitwerk; let file paths dictate the constant namespace; `setup` once, no runtime `eager_load`.
- One class or module per file, named after the constant it defines.
- Define nested constants with the full `module/class` nesting form, not compact path syntax.
- No load-time side effects on `require`; only definitions, lazy/explicit initialization.
- Follow the standard gem layout: single entry point, `sig/`, `test/`, one gemspec.
- `require_relative` for in-project files outside autoload, `require` for external gems; sort and group.
- No top-level constant pollution; everything under the gem's namespace module.
- Keep the dependency graph acyclic; extract the shared abstraction to break a cycle, never lazy-require.
- Re-export the public contract from the top-level namespace; hide internals under `Internal`.
- Group files by feature domain, not by technical layer; keep `shared/` thin.

### 13 — Resource Management

- Use block form for every closable resource; nest acquisitions for correct teardown order.
- When no block form exists, acquire then release in `ensure`; make close idempotent and nil-guarded.
- Size connection pools with a named constant; never one connection per request; pair with a checkout timeout.
- Bound every cache by size or TTL; an unbounded memo is a memory leak; synchronize shared caches.
- Set a deadline on every external I/O call via the client's own timeout option, not `Timeout.timeout`.
- Never rely on finalizers or the GC for cleanup; `define_finalizer` only as a documented leak detector.
- Release resources in reverse acquisition order; teardown is exception-safe and idempotent.
- Join or shut down every thread, Ractor, and Fiber on exit; worker pools expose `shutdown`.
- Use `at_exit` sparingly and idempotently; prefer explicit lifecycle ownership.
- Wrap a raw resource in a small object that owns its lifecycle and exposes a `with_` block API.

### 14 — Documentation

- YARD every public class and method; exempt only the privately visible and obvious.
- Never restate a `sig` in prose; YARD adds intent, constraints, units, sentinels, edge cases.
- Why-comments explain reasoning, not mechanics.
- A file or class header states what it is and how to use it; multi-class or class-less files get a top-of-file comment.
- Put `@example` on every non-obvious public API.
- Format TODOs as `# TODO(Full Name): explanation`.
- Never leave commented-out code; delete it — `git` remembers.
- Use `#` line comments only; never `=begin`/`=end`.
- Keep comments in sync with the code; delete one you cannot maintain.
- Write comments as narrative: full sentences, proper capitalisation and punctuation.

### 15 — Performance

- Measure first; profile the hot path with `stackprof`/`memory_profiler`/`benchmark-ips`; never touch a cold path.
- Optimize the slowest resource first: network > disk > memory > CPU.
- Ship under a JIT (YJIT) and write monomorphic call sites it compiles well; benchmark with the JIT enabled.
- Assign every instance variable in `initialize`, in a consistent order, for stable object shapes.
- Freeze string literals; reuse buffers; allocate less — hoist allocations out of hot loops.
- Build strings with `<<`; never `String#+` in a loop; freeze the result at the boundary.
- Use `.lazy` for large, streamed, or infinite sequences; not for small finite collections.
- Prefer `size` over `count`/`length` where equivalent; `count` can trigger a query.
- Use the most specific string method; avoid `gsub`/regex when `sub`/`tr`/`delete`/`start_with?` suffice.
- `Hash#fetch` with a default block; `Hash.new` with a factory block for cached defaults.
- Batch over per-row work; never issue N+1 queries or requests; eager-load associations.
- Prefer symbols over strings for keys and labels; convert external strings once at the boundary.
