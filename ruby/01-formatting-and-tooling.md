# 01 — Formatting & Tooling

Formatting is not a matter of taste. We delegate all decisions to `rubocop` + `rubocop-airbnb` and spend judgment on things a cop cannot check. This chapter fixes the Ruby version pin, the toolchain, the mandatory file-header comments, and the mechanical whitespace shape every dexpace Ruby file inherits before a line of domain code is written.

## What good looks like

```ruby
# frozen_string_literal: true
# typed: strict

require "sorbet-runtime"

module Ecommerce
  class OrderCalculator
    extend T::Sig

    sig { params(order: Order, as_of: Time).returns(Money) }
    def self.total(order, as_of:)
      raise ArgumentError, "order has no lines" if order.lines.empty?

      subtotal = order.lines.reduce(Money.zero) do |sum, line|
        sum + line.unit_price * line.quantity
      end

      tax = TaxTable.fetch(order.region).apply(subtotal, as_of: as_of)
      total = subtotal + tax

      raise "total cannot be less than subtotal" if total < subtotal

      total
    end
  end
end
```

The file opens with both magic comments separated from the `require` by a blank line (1.4). The Ruby version is ≥ 4.0, so frozen string literals are the default behaviour, but we still declare the comment explicitly — it documents intent, not just compiler behaviour (1.4). Sorbet is wired in via `extend T::Sig` and a `sig` block on the public method (1.3). Two-space indentation, no hard tabs (1.5), columns well under 100 (1.6), double-quoted strings throughout (1.7), one expression per line, no semicolons (1.8). The multiline `reduce` block places one element per conceptual unit and the closing `end` aligns with the opening call (1.10). Metric cops in config would reject this method if it grew past 25 lines or nested past three levels (1.11).

## Rules

### 1.1 — Pin Ruby ≥ 4.0 via `.ruby-version`; commit `Gemfile.lock`.

**Reasoning, step by step:**
1. `.ruby-version` is read by `rbenv`, `chruby`, and `asdf` alike. One file, one pin, zero per-developer negotiation. Without it, each machine resolves whatever the system Ruby happens to be and bugs appear only in CI.
2. Ruby 4.0 is the hard floor because it ships frozen string literals as a default, tightens the object model, and is the baseline Sorbet supports for the target strictness. Lower versions are not a reviewed project choice — they are an omission.
3. `Gemfile.lock` pins the entire resolved dependency graph. Commit it: every developer and every CI node installs the same versions. An uncommitted lockfile makes the build non-reproducible and turns a transitive bump into a silent production change.
4. In CI, run `bundle install --frozen` to reject any lockfile drift caused by an unreviewed `bundle update`. A drift that reaches main blocks the whole team on the next pull.

**Enforcement:** presence of `.ruby-version` containing `4.x` or higher; `Gemfile.lock` committed; `bundle install --frozen` in CI pipeline.

### 1.2 — `rubocop` + `rubocop-airbnb` is the baseline; formatting is non-negotiable.

**Reasoning, step by step:**
1. `rubocop-airbnb` bundles the Airbnb cop set on top of RuboCop core. Inheriting it in `.rubocop.yml` with `inherit_gem: rubocop-airbnb: ...` means every project starts from the same baseline and local `.rubocop.yml` is the narrowest possible diff above that.
2. Pre-commit and CI both run `bundle exec rubocop --format progress`. A formatter failure blocks the commit and blocks the merge. Reviewers never comment on formatting; if it merged, it passed the cop.
3. Any cop override in `.rubocop.yml` must carry a comment naming the chapter and rule that recorded the deviation. An unexplained `Enabled: false` is a deviation without a ledger entry and is rejected in review.
4. `rubocop --autocorrect-all` is the only sanctioned way to fix style: it applies all safe autocorrections at once. Running it in pre-commit prevents style drift from building up.

Worked example:

```ruby
# .rubocop.yml
inherit_gem:
  rubocop-airbnb: .rubocop.yml   # chapter 01 baseline

AllCops:
  NewCops: enable
  TargetRubyVersion: 4.0

# deliberate deviation — see deviations ledger in README.md
Style/StringLiterals:
  EnforcedStyle: double_quotes
```

**Enforcement:** `RuboCop/InheritedMethods`; `bundle exec rubocop` in pre-commit hook and CI; review rejects unexplained cop overrides.

### 1.3 — Sorbet is wired in: `sorbet-static` + `sorbet-runtime`; `srb tc` is a CI gate.

**Reasoning, step by step:**
1. `sorbet-static` is the offline type-checker; `sorbet-runtime` enforces `sig` contracts at runtime. Both are required — static analysis catches shape errors at commit time; runtime enforcement catches them in tests and staging before they reach production.
2. `srb tc` runs in CI as a hard gate, the same way `rubocop` does. A new file that fails the typechecker blocks the merge; a file that degrades an existing sigil to `# typed: false` without a ledger entry also blocks. The gate is the guarantee that the type coverage floor only ever rises.
3. New files are `# typed: strict` minimum. `strict` requires `sig` on every method and treats missing return types as errors. Lower sigils — `# typed: true`, `# typed: false`, `# typed: ignore` — are permitted only for legacy bridges that have not yet been migrated, and every such file must be recorded in a migration tracking list in the repo.
4. Runtime checking means a `sig` is not documentation — it is an executed assertion. A method whose runtime argument violates its `sig` raises `TypeError` immediately, surfacing bugs at the boundary rather than several call frames later.

**Enforcement:** `srb tc` in CI; `# typed: strict` required in new files enforced by a custom RuboCop cop (`Sorbet/StrictSigil`) or a pre-commit check; legacy sigil exceptions recorded in `docs/sorbet-migration.md`.

### 1.4 — Mandatory magic comments: `# frozen_string_literal: true` and `# typed:` sigil, separated from code by a blank line.

**Reasoning, step by step:**
1. `# frozen_string_literal: true` must appear as the very first line of every `.rb` file, even though Ruby 4.0 enables freezing by default. The explicit comment documents intent, survives a downgrade of the runtime setting, and satisfies the `Style/FrozenStringLiteralComment` cop that enforces it mechanically.
2. The `# typed:` sigil must appear immediately after on the second line. Sorbet's own scanner requires the sigil in the first five lines; placing it second is both spec-compliant and makes the two mandatory comments form a visible pair at a glance.
3. A blank line after the comment block separates toolchain declarations from human-readable content — requires, module/class headers, or YARD doc. Without it, the comments read as part of the class header and the distinction is lost.
4. No other magic comments belong at the top. `# encoding: utf-8` is redundant since Ruby 2.0; `# warn_indent: true` is a local debugging aid that must not be committed.

Worked example:

```ruby
# frozen_string_literal: true
# typed: strict

# Represents a line item on a customer order.
class LineItem
  extend T::Sig
  # ...
end
```

**Enforcement:** `Style/FrozenStringLiteralComment: EnforcedStyle: always` (RuboCop); `Sorbet/StrictSigil` or `srb tc`; review rejects any additional magic comments.

### 1.5 — 2-space soft tabs, Unix LF, UTF-8, final newline, no trailing whitespace.

**Reasoning, step by step:**
1. Two-space indentation is the Ruby community consensus, codified by rubystyle.guide and the Airbnb baseline. Hard tabs render at different widths in every tool and are banned; `Tab` cop in RuboCop rejects them automatically.
2. Unix LF line endings (`\n`) prevent CRLF from leaking in on Windows contributors and causing spurious diffs. `.gitattributes` sets `*.rb text eol=lf` so the conversion is at the repo boundary, not per-developer.
3. UTF-8 is the only permitted source encoding; Ruby 3+ defaults to it. Declaring `# encoding: utf-8` is therefore redundant and must not appear (rule 1.4).
4. A final newline is required because POSIX defines a text file as a sequence of newline-terminated lines; editors and `git diff` both produce noise without it. Trailing whitespace on any line is removed; it is invisible, creates spurious diffs, and no tool depends on it.

**Enforcement:** `Layout/IndentationStyle`, `Layout/TrailingWhitespace`, `Layout/EndOfLine` (RuboCop); `.gitattributes` for LF; editor config via `.editorconfig` `insert_final_newline = true`.

### 1.6 — Line length ≤ 100 columns; the Airbnb bound is deliberate.

**Reasoning, step by step:**
1. One hundred columns is the Airbnb canonical cap. Shopify uses 120; we take Airbnb's tighter bound deliberately because narrower lines survive side-by-side diffs, two-pane editors, and code review on a laptop without horizontal scrolling.
2. The cap forces decomposition before it forces wrapping. A method call that would hit 101 columns almost always benefits from an extracted local name or a multi-line form rather than a continuation line.
3. Comments and YARD strings count toward the limit. Wrapping a long URL is the one accepted exception — place the URL alone on its own line rather than wrapping mid-token.

Worked example:

```ruby
# Good — extracted variable keeps both lines under 100
discount_amount = DiscountEngine.calculate(order, coupon: promo_code, as_of: Time.now.utc)
total = order.subtotal - discount_amount

# Bad — single expression that runs past 100 columns
total = order.subtotal - DiscountEngine.calculate(order, coupon: promo_code, as_of: Time.now.utc)
```

**Enforcement:** `Layout/LineLength: Max: 100` (RuboCop); review rejects `# rubocop:disable Layout/LineLength` except for lone URLs.

### 1.7 — Double-quoted strings always.

**Reasoning, step by step:**
1. One rule, zero per-literal decisions. The question "should this string be single- or double-quoted?" costs mental cycles that could go to correctness. The answer is always double quotes.
2. Double quotes are the Shopify default and our recorded deviation from Airbnb's silence on the matter. The deviation is captured in the deviations ledger (README.md) and enforced by `Style/StringLiterals: EnforcedStyle: double_quotes`.
3. Interpolation works in double-quoted strings without any edit. A single-quoted string that grows to need interpolation requires two-character surgery on both delimiters. Starting with double quotes eliminates the friction.
4. `%q()`, `%Q()`, and `%{}` are banned for ordinary strings. They introduce a third quoting syntax with no benefit over double quotes and make the code harder to grep.

**Enforcement:** `Style/StringLiterals: EnforcedStyle: double_quotes` (RuboCop); `Style/StringLiteralsInInterpolation: EnforcedStyle: double_quotes`.

### 1.8 — One expression per line; no semicolons except single-line class bodies.

**Reasoning, step by step:**
1. A semicolon on one line is two statements the reader may miss. Separate statements belong on separate lines; the line break is the separator the Ruby parser and every reader expect.
2. The one permitted exception is a single-line class body for an empty class or a minimal shell: `class NotFoundError < StandardError; end`. This is idiomatic, instantly parseable, and saves three lines. A class body with even one method breaks out to full multi-line form.
3. No other single-line compounds: no `def foo; bar; end`, no `if cond; x; end`. The full multi-line form is always permitted and never ambiguous.

Worked example:

```ruby
# Good — one expression per line
sku = Sku.new(code: "ABC-123")
price = Inventory.fetch(sku).unit_price

# Good — permitted single-line exception
class InventoryError < StandardError; end

# Bad — semicolon compounding
sku = Sku.new(code: "ABC-123"); price = Inventory.fetch(sku).unit_price
```

**Enforcement:** `Style/Semicolon` (RuboCop); `Style/OneLineConditional`.

### 1.9 — Spaces around operators, after punctuation, inside braces; no space inside brackets or after unary operators.

**Reasoning, step by step:**
1. Spaces around binary operators (`+`, `-`, `*`, `/`, `=`, `==`, `<`, `>`, `&&`, `||`, `=>`) make the operands visually distinct. Omitting them collapses expressions into strings of characters that require active parsing: `a=b+c` forces a different cognitive mode than `a = b + c`.
2. Space after every comma, colon (in argument lists and hash literals), and semicolon. No space before any of them. This is the universal rule across Ruby, Python, and C-family languages because it matches English prose spacing and trains a consistent expectation.
3. Space inside `{ }` in hash literals and blocks: `{ key: value }`, `array.map { |x| x * 2 }`. No space immediately inside `[]` or `()`: `array[0]`, `method(arg)`. This distinguishes block/hash delimiters from indexing/call delimiters at a glance.
4. No space after unary `!`, `~`, or `+`: `!valid?`, not `! valid?`. No space in range literals: `1..10`, not `1 .. 10`. No space in the lambda literal: `->(x, y) { x + y }`, not `-> (x, y) { x + y }`.

Worked example:

```ruby
# Good
total = subtotal + tax
items = order.lines.select { |line| line.quantity > 0 }
config = { timeout_ms: 5_000, retries: 3 }
range = 1..100
validator = ->(amount) { amount >= 0 }

# Bad
total=subtotal+tax
items = order.lines.select {|line| line.quantity > 0}
config = {timeout_ms: 5_000, retries: 3}
```

**Enforcement:** `Layout/SpaceAroundOperators`, `Layout/SpaceAfterComma`, `Layout/SpaceInsideHashLiteralBraces`, `Layout/SpaceInsideParens`, `Layout/SpaceInsideBrackets`, `Layout/SpaceAfterNot`, `Layout/SpaceAroundRangeLiterals` (RuboCop).

### 1.10 — Multiline literals and calls: one element per line, trailing comma, closing delimiter on its own line; chains break with a leading dot.

**Reasoning, step by step:**
1. One element per line makes additions, removals, and reorders produce single-line diffs. A three-element array on one line produces a three-line diff when one element changes; the same array split across lines produces a one-line diff.
2. A trailing comma after the last element is mandatory in multiline array, hash, and argument list literals. Without it, adding an element to the end requires editing two lines (the old last element and the new one), breaking the single-line diff property.
3. The closing delimiter — `]`, `}`, `)`, or `end` — goes on the line after the last element, dedented to the level of the opening line. This is the only position that keeps indent level visually consistent and lets `rubocop` verify alignment automatically.
4. Method chains longer than one receiver-plus-call break at each dot with the dot leading on the continuation line, indented one level. A trailing dot on the prior line is invisible at a glance and suggests the expression is complete when it is not.

Worked example:

```ruby
# Good — multiline array with trailing comma
line_items = [
  LineItem.new(sku: "SKU-1", quantity: 2),
  LineItem.new(sku: "SKU-2", quantity: 1),
  LineItem.new(sku: "SKU-3", quantity: 4),
]

# Good — method chain with leading dots
summary = order
  .lines
  .select { |line| line.active? }
  .map { |line| line.unit_price * line.quantity }
  .reduce(Money.zero, :+)

# Bad — trailing dot
summary = order.
  lines.
  select { |line| line.active? }
```

**Enforcement:** `Style/TrailingCommaInArrayLiteral`, `Style/TrailingCommaInHashLiteral`, `Style/TrailingCommaInArguments` (set to `consistent_comma`); `Layout/MultilineMethodCallIndentation: EnforcedStyle: indented`; `Layout/DotPosition: EnforcedStyle: leading` (RuboCop).

### 1.11 — Metric caps live in config and back chapter 05: MethodLength 25, ParameterLists 4, BlockNesting 3, AbcSize, CyclomaticComplexity.

**Reasoning, step by step:**
1. Caps that live only in prose are ignored; caps that live in `.rubocop.yml` are enforced. Every metric limit is expressed as a RuboCop cop configuration so the boundary is mechanical, not a matter of reviewer memory.
2. `Metrics/MethodLength Max: 25` with `CountAsOne: [array, hash, heredoc]` is the Ruby-scaled sibling of Go's 70 and Kotlin's 60. The `CountAsOne` exemptions prevent a method whose body is a well-structured hash literal from looking artificially long — the literal is one logical unit and reads as one. Aim for 5–15 lines; 25 is the hard ceiling, not the target.
3. `Metrics/ParameterLists Max: 4` matches Tiger Style's limits-on-everything principle. A method requiring a fifth argument is almost always doing too much or its arguments belong in a typed value object. The cop forces the conversation at write time, not in review.
4. `Metrics/BlockNesting Max: 3` applies the same pressure as `max-depth` in the TypeScript guide. Guard clauses keep the happy path flush left and keep nesting shallow; a fourth level almost always indicates a missing extraction.
5. `Metrics/AbcSize` and `Metrics/CyclomaticComplexity` catch complexity that line count misses — a 10-line method with 8 branches has more cognitive weight than a 20-line straight-line sequence. Leave the defaults unless a specific exception is recorded.

Worked example:

```yaml
# .rubocop.yml excerpt
Metrics/MethodLength:
  Max: 25
  CountAsOne:
    - array
    - hash
    - heredoc

Metrics/ParameterLists:
  Max: 4

Metrics/BlockNesting:
  Max: 3
```

**Enforcement:** `Metrics/MethodLength`, `Metrics/ParameterLists`, `Metrics/BlockNesting`, `Metrics/AbcSize`, `Metrics/CyclomaticComplexity` (RuboCop); cop config committed in `.rubocop.yml`; review rejects inline `rubocop:disable Metrics/` without a ledger entry.

## Cross-references

- Method length cap and guard-clause discipline: [05-methods.md](./05-methods.md).
- `# typed: strict`, `sig` on every method, and Sorbet runtime enforcement: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- Naming conventions the formatter leaves untouched — `snake_case`, predicate `?`, effect verbs: [02-naming-conventions.md](./02-naming-conventions.md).
- YARD documentation comments and why-comments that live after the magic-comment block: [14-documentation.md](./14-documentation.md).
- `Data.define` value objects and `class << self` grouping that interact with method-length counting: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Frozen values, immutable collections, and `freeze` on constants: [04-variables-and-declarations.md](./04-variables-and-declarations.md).
- Module organization, Zeitwerk autoloading, and one-class-per-file that the file-header shape enables: [12-module-organization.md](./12-module-organization.md).
