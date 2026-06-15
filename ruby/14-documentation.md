# 14 — Documentation

YARD is the documentation layer for public Ruby API; a `sig` is the type layer. Keep them separate: the signature states the types, the comment states the contract, the why, and the edge cases. A comment that cannot be maintained must be deleted — a stale comment is an active lie.

## What good looks like

A fully documented public method: a why-summary, `@param` only where the name and sig fall short, the failure modes, and one runnable example.

```ruby
# frozen_string_literal: true
# typed: strict

module Inventory
  # Reserves `quantity` units of `sku` against the caller's order, holding them
  # for {RESERVATION_TTL_SECONDS} so checkout can complete without a race.
  #
  # The hold is best-effort: stock can still sell out between reservation and
  # capture, which is why {#capture_reservation} re-checks. Prefer this over
  # decrementing stock directly — a raw decrement leaks units when an order is
  # abandoned.
  #
  # @param quantity [Integer] units to hold; must be a positive whole-unit count
  # @raise [OutOfStockError] fewer than `quantity` units are available — show the back-in-stock prompt
  # @raise [SkuRetiredError] the SKU is no longer sold — drop it from the order
  # @example
  #   hold = Inventory.reserve(order_id: order.id, sku: "SKU-1024", quantity: 2)
  #   Inventory.capture_reservation(hold.id)
  sig { params(order_id: Integer, sku: Sku, quantity: Integer).returns(Reservation) }
  def self.reserve(order_id:, sku:, quantity:)
    # ...
  end
end
```

The summary explains *why* a caller reaches for `reserve` over a raw decrement (14.3). `@param` appears only on `quantity` because that is the argument whose constraint — positive, whole-unit integer — the `sig` cannot capture (14.2). Both failure modes tell the caller what to do next (14.1). The `@example` is a real, runnable call site (14.5). The prose says nothing the `sig` already says — it adds the *why* and the edge case, nothing else.

## Rules

### 14.1 — YARD every public class and method; exempt only the privately visible and obvious.

**Reasoning, step by step:**
1. The audience for a public symbol is a caller who will never open its source; they get the signature and the YARD hover-card. If the contract is absent from YARD, it does not exist for them.
2. A not-externally-visible method whose name already carries the full meaning — `valid?`, `to_s`, a two-line predicate — does not need prose that would only restate the name. Add YARD when a reader needs more than the name and `sig` to use the method safely.
3. A public class carries a class-level YARD comment: what it models, when to reach for it, and what invariants it guarantees. A `Data.define` value object without a class comment leaves callers guessing at the fields' semantics.

**Enforcement:** review; every public method and class exported from the module surface carries a YARD block, verified during code review of new API additions.

### 14.2 — Never restate a `sig` in prose.

**Reasoning, step by step:**
1. `@param order_id [Integer] the order id` adds nothing to `order_id: Integer` in the `sig`; the name and the type already say it. Now there are two places to update on a rename.
2. A `sig` enforced at runtime is a stronger statement than prose; prose that duplicates it is noise at best and a confident lie after a refactor at worst.
3. YARD earns its line by adding what the `sig` cannot: intent, constraints, units, valid ranges, the meaning of a sentinel, an edge case the type system cannot express. "Must be a positive whole-unit count" adds information; "an Integer" does not.

**Enforcement:** review; reject `@param` and `@return` lines that only echo the type; keep the ones that add a constraint, unit, or sentinel.

### 14.3 — Why-comments explain reasoning, not mechanics.

**Reasoning, step by step:**
1. `# increment retry counter` above `retries += 1` is dead weight — the line says that already. `# back off only on 5xx; 4xx will never succeed on retry` carries a decision the code cannot express.
2. The *what* is verifiable by reading the code; the *why* is not. Spend comment budget on the invisible part: the constraint that forced this shape, the bug this guards against, the spec clause behind the magic constant.
3. Assume the reader knows Ruby but not your domain and not your intent. They can parse `line_items.select(&:taxable?)` without a comment; they cannot know that non-taxable items are still included in the fulfillment fee unless you say so.

```ruby
# Select only taxable items for the sales-tax line; the fulfillment fee applies
# to all line items and is computed separately in FulfillmentCalculator.
taxable = order.line_items.select(&:taxable?)
```

**Enforcement:** review; delete mechanics-narrating comments on sight, keep the ones a reviewer could not have inferred from the diff.

### 14.4 — A file or class header states what it is and how to use it; a file with zero or more than one class gets a top-of-file comment.

**Reasoning, step by step:**
1. A module file with multiple classes or no class at all gives a reader no entry point without a file-level comment. The comment is the map before the territory.
2. A class-level header states the model: what the class represents, its invariants, and the pattern a caller uses. A `Customer` class without a header leaves callers wondering whether it is a value object, a service object, or an ActiveRecord subclass.
3. Airbnb's guide mandates a top-of-file comment when a file contains zero or more than one class. Follow that rule exactly; a single-class file where the class header comment already says everything is exempt from a separate file-level comment.

```ruby
# frozen_string_literal: true
# typed: strict

# Utilities for normalising and validating Money amounts at order ingestion.
# All methods are pure functions: no I/O, no mutation of arguments.
# Use {MoneyParser.parse} at every inbound boundary before arithmetic.
module MoneyUtils
  # ...
end
```

**Enforcement:** rubocop-airbnb; review confirms the comment addresses what the file is and how to use it, not just what it contains.

### 14.5 — Put `@example` on every non-obvious public API.

**Reasoning, step by step:**
1. A signature shows the shape of a call but not the shape of usage — the order of operations, what to do with the return, which helper pairs with it. One worked example answers all three faster than prose.
2. A runnable example is the fastest thing a caller reads and the hardest thing to misread; prose describing the same call leaves room for ambiguity the code does not.
3. Obvious one-liners — a pure `Money.zero` or a predicate `order.paid?` — do not need an example; the signature is the example. Reserve `@example` for APIs with non-obvious sequencing, setup, or pairing.

```ruby
# @example
#   result = OrderPricer.price(
#     order: order,
#     customer: customer,
#     effective_at: Time.now.utc,
#   )
#   puts result.total_cents
```

**Enforcement:** review; non-obvious public methods carry a realistic `@example` that reflects the current API.

### 14.6 — Format TODOs as `# TODO(Full Name): explanation`.

**Reasoning, step by step:**
1. An anonymous `# TODO` is a wish nobody owns. `# TODO(Lena Schmidt): remove after SKU normaliser ships in v4` is a task with a name, a reason, and enough context to evaluate later.
2. The full name makes the comment `grep`-able by person: `grep -r "TODO(Lena" .` finds every outstanding item in one command. A first name alone or a username alias breaks across team members.
3. The explanation is mandatory: "why does this debt exist" and "what removes it" must both fit in the comment. A TODO without an explanation is a note that says "fix this" and nothing more — insufficient to act on or to decide whether it is still relevant.

```ruby
# TODO(Omar Mazari): replace Money#cents with MoneyV2#minor_units once
# the currency-migration rake task has been run in production (see ADR-0042).
```

**Enforcement:** review; rubocop custom cop or grep in CI rejects `# TODO` not matching `# TODO\([^)]+\): .+`.

### 14.7 — Never leave commented-out code; delete it.

**Reasoning, step by step:**
1. Commented-out code is the loudest kind of clutter: it looks like it might matter, forces every reader to parse it and decide whether to restore it, and provides no explanation of why it was removed.
2. Version control remembers every line ever committed. `git log -S 'removed_method'` recovers it in seconds. The comment buys nothing that `git` does not already provide — and `git` does not mislead.
3. If the code must be recoverable for a specific reason, the reason belongs in a commit message or an ADR, not in a comment block.

**Enforcement:** rubocop `Style/CommentedKeyword` and review; commented-out code is rejected at review regardless of apparent intent.

### 14.8 — Use `#` line comments only; never `=begin`/`=end`.

**Reasoning, step by step:**
1. `=begin`/`=end` block comments cannot be indented — they must start in column 0. Inside a method or class body they look like a syntax error to anyone who has not memorised the rule.
2. `#` line comments work at any indentation level, are easy to spot with `grep`, and compose naturally with the rest of the file. There is no advantage `=begin`/`=end` holds that YARD or `#` lines do not already cover.
3. YARD uses `#` comment lines for documentation blocks. Mixing block-comment syntax into a codebase that uses YARD creates two documentation dialects with no benefit.

**Enforcement:** rubocop `Style/BlockComments`; CI rejects any `=begin` outside an intentional literal-string test fixture.

### 14.9 — Keep comments in sync with the code; delete one you cannot maintain.

**Reasoning, step by step:**
1. A reader trusts a comment more than they trust their own reading of the code. A wrong comment spends that trust to point them at the wrong conclusion; deleting it strictly improves the file.
2. Documentation drifts only when a change updates the code and leaves the prose behind. The gap costs nothing to close at edit time and is expensive forever after, once nobody remembers which of the two is right.
3. "I'll update the comment later" is technical debt that never gets paid. The discipline is one commit: code and its comment move together, or not at all. This is root rule 12 — zero debt — applied to prose.

**Enforcement:** review; a diff that changes behaviour and leaves surrounding YARD or inline comments stale does not merge — update or delete in the same commit.

### 14.10 — Write comments as narrative: full sentences, proper capitalisation and punctuation.

**Reasoning, step by step:**
1. A comment that reads like a headline ("Retry logic") gives less information than a sentence ("Retry up to three times with exponential back-off; give up on the fourth failure and raise."). Sentences force completeness.
2. Proper capitalisation and ending punctuation signal that the comment is finished prose, not a placeholder. A fragment like `# handles nil` looks like a stub; `# Returns zero when the order has no line items.` reads as intentional documentation.
3. The single exception is a short trailing note on the same line as code — `total = subtotal + tax  # cents` — where a full sentence would overflow the 100-column limit. That note does not need terminal punctuation.

```ruby
# Bad — fragment, no punctuation, looks like a stub.
# check for abandoned carts

# Good — sentence, tells the reader exactly what and why.
# Skip orders with no activity in the last 30 days; the warehouse
# considers them abandoned and has already released their stock holds.
order.line_items.select { |li| li.updated_at > 30.days.ago }
```

**Enforcement:** review; YARD blocks and standalone inline comments are complete sentences; trailing same-line notes are exempt from the sentence requirement.

## Cross-references

- Sorbet `sig` blocks that replace type prose in YARD: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- Naming as the first line of documentation — predicate `?`, bang `!`, effect-verb discipline: [02-naming-conventions.md](./02-naming-conventions.md).
- Public method surface and the minimal API contract that YARD documents: [10-api-design.md](./10-api-design.md).
- `@raise` and the `StandardError` subclass hierarchy YARD references: [08-error-handling.md](./08-error-handling.md).
- One class per file and Zeitwerk module layout that the file-header rule (14.4) assumes: [12-module-organization.md](./12-module-organization.md).
- Formatting rules — 100-col limit, double quotes, 2-space indent — that apply inside YARD fences and examples: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
