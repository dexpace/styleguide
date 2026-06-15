# 12 — Module Organization

How code is grouped into files, how files are grouped into directories, and how a gem exposes itself to callers. The shape of the constant graph is the first thing a new contributor reads and the last thing a refactor can cheaply change — it earns the same care as the type signatures. This chapter is about the boundaries between files; chapter 10 is about the API those boundaries publish.

## What good looks like

```text
gems/commerce/
├── commerce.gemspec
├── lib/
│   ├── commerce.rb                     # entry point: requires, re-exports public surface
│   └── commerce/
│       ├── checkout/
│       │   ├── order.rb                # Commerce::Checkout::Order
│       │   ├── line_item.rb            # Commerce::Checkout::LineItem
│       │   └── pricing_calculator.rb   # Commerce::Checkout::PricingCalculator
│       ├── inventory/
│       │   ├── sku.rb                  # Commerce::Inventory::Sku
│       │   └── stock_level.rb          # Commerce::Inventory::StockLevel
│       ├── shared/
│       │   └── money.rb                # Commerce::Shared::Money
│       └── internal/
│           └── discount_engine.rb      # Commerce::Internal::DiscountEngine — not public
├── sig/
│   └── commerce.rbi                    # Sorbet signatures
└── test/
    └── commerce/
        └── checkout/
            └── order_test.rb
```

```ruby
# frozen_string_literal: true
# typed: strict

# lib/commerce.rb — entry point; only requires and public re-exports here.

require "zeitwerk"

loader = Zeitwerk::Loader.for_gem
loader.setup

module Commerce
  extend T::Sig

  # Re-export the public contract; Internal is intentionally omitted.
  Order     = Checkout::Order
  LineItem  = Checkout::LineItem
  Sku       = Inventory::Sku
  Money     = Shared::Money
end
```

```ruby
# frozen_string_literal: true
# typed: strict

module Commerce
  module Checkout
    class Order
      extend T::Sig

      sig { returns(T::Array[LineItem]) }
      attr_reader :line_items

      sig { params(line_items: T::Array[LineItem]).void }
      def initialize(line_items:)
        @line_items = T.let(line_items, T::Array[LineItem])
      end
    end
  end
end
```

The tree groups by domain (`checkout/`, `inventory/`) rather than technical kind (12.10). Zeitwerk enforces the file-to-constant mapping — `commerce/checkout/order.rb` defines exactly `Commerce::Checkout::Order` (12.1). Each file contains one class (12.2). `Order` is defined with a nested `module Commerce; module Checkout` block, not the compact `Commerce::Checkout::Order` form (12.3). The entry point does no work — it wires the loader and re-exports names (12.4, 12.9). `Commerce::Internal` stays out of the public aliases (12.9).

## Rules

### 12.1 — Use Zeitwerk; let file paths dictate the constant namespace.

**Reasoning, step by step:**
1. Zeitwerk is the standard Ruby autoloader (Rails 6+, Bundler-compatible for gems). It maps `lib/commerce/checkout/order.rb` to `Commerce::Checkout::Order` with zero explicit `require` calls inside the namespace.
2. The convention is the contract: a wrong filename is a `NameError` at first use, not a silent shadow. The loader surfaces the mistake the moment the constant is needed.
3. Consistent mapping eliminates the "where does this constant live?" question. File path _is_ the answer — no hunting through require chains.
4. Call `Zeitwerk::Loader.for_gem` in the entry point; call `loader.setup` once. Do not call `loader.eager_load` at runtime — that is a deployment or test-setup concern.

Worked example:
```ruby
# lib/commerce.rb
require "zeitwerk"
loader = Zeitwerk::Loader.for_gem
loader.setup
```

`Commerce::Checkout::Order` is available without a single `require` inside the namespace directory.

**Enforcement:** `Zeitwerk::Loader#verify!` in CI (`loader.eager_load; loader.verify!`); a filename that doesn't match its constant raises on load.

### 12.2 — One class or module per file, named after the constant it defines.

**Reasoning, step by step:**
1. Zeitwerk's convention assumes it. Two constants in one file silently leaves one unloadable; the loader only triggers on the filename it sees.
2. "What is in this file?" must have a one-word answer. A file that defines `Order` and `LineItem` together answers "both" — which means neither is the obvious home.
3. One constant per file makes `git blame` exact: every line of `Commerce::Order` lives in `order.rb`. Mixed-constant files scatter blame and make diffs harder to read.
4. The only sanctioned exception: a class-level private struct or `Data.define` used nowhere but that file may be declared inside the same file without its own file.

**Enforcement:** `Zeitwerk::Loader#verify!` (mismatched constants raise); review rejects multi-constant top-level files.

### 12.3 — Define nested constants with the full `module/class` nesting form, not compact path syntax.

**Reasoning, step by step:**
1. The compact form `class Commerce::Checkout::Order` opens the constant without opening each intermediate namespace module. Ruby then resolves un-prefixed names against the lexical nesting — which is only the file's top-level scope — not the intermediate modules. A constant visible inside `Commerce` is invisible inside the compacted class body.
2. This is a documented Shopify rule and a frequent source of subtle `NameError`s: the bug appears only when the intermediate module holds a constant the inner class references, and only when autoload timing lands wrong.
3. The nested form is explicit about what scope each body executes in. `module Commerce; module Checkout; class Order` — each keyword opens one level and the reader sees each ancestor in the source.
4. Nested blocks require more indentation but that cost is paid in safety and grep-ability: `grep -r "module Commerce"` locates every file that participates in that namespace.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# Bad — compact form; constant lookup skips intermediate modules.
class Commerce::Checkout::Order
  TAX_RATE = Shared::TAX_RATE  # NameError: Commerce::Shared not in lexical scope
end

# Good — nested form; each module is open and in the lexical scope.
module Commerce
  module Checkout
    class Order
      TAX_RATE = Shared::TAX_RATE  # resolves Commerce::Shared::TAX_RATE correctly
    end
  end
end
```

**Enforcement:** RuboCop `Style/ClassAndModuleChildren` set to `nested`; review rejects compact-form definitions.

### 12.4 — No load-time side effects on `require`: only definitions.

**Reasoning, step by step:**
1. Importing a file must be free. A network call, a database query, a global registry mutation, or a `puts` at file scope makes load order load-bearing — behavior changes based on which file Zeitwerk happens to load first.
2. Side-effectful requires are invisible at call sites. The reader of `order.rb` has no way to know that requiring it opens a socket, and no way to suppress it in tests.
3. They break Zeitwerk's lazy-load model: Zeitwerk defers constant loading to first use. A file with a side effect fires that effect at an unpredictable point during the request lifecycle.
4. If initialization is needed — registering a plugin, seeding a cache — expose an explicit `Commerce.configure { }` block or a `Commerce::Loader.setup!` method. Callers invoke it once, intentionally, in an initializer.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# Bad — global mutation at load time.
module Commerce
  RATES = TaxService.fetch_rates  # network call on require
end

# Good — lazy, explicit initialization.
module Commerce
  extend T::Sig

  sig { returns(T::Hash[String, Float]) }
  def self.rates
    @rates ||= TaxService.fetch_rates
  end
end
```

**Enforcement:** review; no top-level expressions other than `module`, `class`, `CONST =`, `require`, `require_relative`, or `extend/include`; CI import-time test suite detects unexpected I/O via `allow_any_instance_of` stubs on `Net::HTTP`.

### 12.5 — Follow the standard gem layout: single entry point, `sig/`, `test/`, one gemspec.

**Reasoning, step by step:**
1. `lib/<gem>.rb` is the one file consumers `require`; it sets up Zeitwerk and re-exports the public surface. Every other file in `lib/` is an implementation detail reached through autoload.
2. `sig/` (or `rbi/`) holds Sorbet-generated RBI files for external gems and handwritten signatures for duck types. Keeping them separate from `lib/` prevents Zeitwerk from accidentally loading them as constants.
3. `test/` mirrors `lib/` in structure: `test/commerce/checkout/order_test.rb` tests `Commerce::Checkout::Order`. Parallel structure means the test for any constant is findable without a search.
4. One `<gem>.gemspec` at the root declares all metadata, runtime dependencies, and development dependencies. Multiple gemspecs in one repo signal the gem should be split.

**Enforcement:** review and gemspec linting (`bundle exec gem specification`); `sig/` is excluded from Zeitwerk paths in `loader.ignore`.

### 12.6 — Use `require_relative` for in-project files outside the autoload path; `require` for external gems; sort and group.

**Reasoning, step by step:**
1. Zeitwerk handles everything under `lib/<gem>/`. The entry point `lib/<gem>.rb` sits one level above and is not autoloaded — it must `require_relative "commerce/version"` or similar manually.
2. External gems are required by name: `require "sorbet-runtime"`. The path is resolved by RubyGems and is stable across installs.
3. Grouping requires into (1) stdlib, (2) external gems, (3) in-project with a blank line between each group mirrors the import discipline in sibling guides. Alphabetical order within each group removes micro-decisions and makes merge conflicts trivial.
4. Never `require_relative` into another gem's internals. If you need a constant from an external gem, `require` the gem's public entry point only.

Worked example:
```ruby
# frozen_string_literal: true
# typed: strict

# lib/commerce.rb

require "json"

require "sorbet-runtime"
require "zeitwerk"

require_relative "commerce/version"
```

**Enforcement:** RuboCop `Require/Sorted` (custom cop or `rubocop-require_tools`); review for missing groups or cross-gem `require_relative`.

### 12.7 — No top-level constant pollution: everything under the gem's namespace module.

**Reasoning, step by step:**
1. A top-level constant (`Order`, `Money`, `Error`) collides with any gem or application constant of the same name. The collision is silent — last one loaded wins — and may surface only in production where load order differs from development.
2. `Commerce::Order` is unambiguous regardless of what other gems define. The namespace is the gem's claim on a name; constants outside it are squatters in a shared yard.
3. Reopening `Object` or `Kernel` to add helper methods is the same violation in method form. Use a module of functions inside the namespace and mix it in where needed.
4. The only sanctioned top-level definition is the gem's namespace module itself (`module Commerce`) in the entry point.

**Enforcement:** RuboCop `Style/TopLevelMethodDefinition`; review rejects constants or methods defined directly in the file without a namespace wrapper.

### 12.8 — The dependency graph must be acyclic; extract the shared abstraction to break a cycle.

**Reasoning, step by step:**
1. Module A requires module B which requires module A — directly or through a chain. Ruby handles some cycles by leaving constants temporarily unresolved, so the failure mode is a `NameError` at runtime, in production, under a load order that differs from development. It works by accident until it doesn't.
2. A cycle is a coupling confession. Two modules that need each other are either one concept split across two files, or they share a third concept that belongs in its own module. Identify that shared concept and extract it.
3. Lazy-requiring to paper over a cycle (`require` inside a method body) hides the coupling instead of resolving it. The graph remains cyclic; now it is invisible.
4. Detect cycles in CI. `bundle exec ruby -e 'require "your_gem"'` with a cycle-detection hook, or inspect the Zeitwerk eager-load order, or use a custom `Bundler::Audit` step. A cycle that reaches CI never reaches production.

Worked example:
```ruby
# Bad — Commerce::Checkout::Order requires Commerce::Inventory::Sku
#        Commerce::Inventory::Sku requires Commerce::Checkout::Order  (cycle)

# Fix — extract the shared type to Commerce::Shared::SkuCode:
# lib/commerce/shared/sku_code.rb  → Commerce::Shared::SkuCode
# Both Order and Sku depend on SkuCode; neither depends on the other.
```

**Enforcement:** CI step running `bundle exec ruby -e 'Zeitwerk::Loader.for_gem.tap { |l| l.setup; l.eager_load }.verify!'`; review rejects lazy `require` calls inside method bodies used to break a cycle.

### 12.9 — Re-export the public contract from the top-level namespace; hide internals under `Internal`.

**Reasoning, step by step:**
1. Callers should import from one place: the gem's top-level module. `Commerce::Order`, not `Commerce::Checkout::Order`. The directory structure is an implementation detail; the public aliases are the promise.
2. The file-path-to-constant mapping is the right internal structure, and it may change. Public aliases in the entry point insulate callers from reorganizations: moving `order.rb` to `fulfilment/order.rb` requires updating one alias, not every caller.
3. Internals that are not part of the promise live under `Commerce::Internal`. Callers who reach into `Commerce::Internal` own every breakage. The name is the warning; no `private_constant` tricks needed (though `private_constant :Internal` adds a runtime guard).
4. This ties directly to chapter 10's minimal-surface principle: the public contract is what you audit, version, and defend. Everything else is noise to a consumer.

Worked example:
```ruby
# lib/commerce.rb

module Commerce
  # Public aliases — the versioned contract.
  Order    = Checkout::Order
  LineItem = Checkout::LineItem
  Sku      = Inventory::Sku
  Money    = Shared::Money

  private_constant :Internal  # runtime guard; callers who dig in own it
end
```

**Enforcement:** review; `private_constant :Internal` in the entry point; any `Commerce::Internal::` reference outside `lib/commerce/internal/` is flagged in review.

### 12.10 — Group files by feature domain, not by technical layer.

**Reasoning, step by step:**
1. A domain folder (`commerce/checkout/`, `commerce/inventory/`) holds every file that participates in that concept — data shapes, behaviour, query logic, errors — side by side. A change to checkout touches one directory.
2. Layer folders (`models/`, `services/`, `repositories/`) scatter one feature across the tree. A checkout change edits four directories; each directory mixes unrelated features that share only a technical role.
3. Change travels together. Domain layout makes a diff local, a code-owner obvious, and deleting a feature `rm -r` on one directory instead of an archaeology dig across four.
4. Cross-domain primitives with no single owner — a `Money` type, a base error class — live under `commerce/shared/`. Keep it thin: every entry there is a coupling thread pulling all domains toward a common centre, and a fat `shared/` is just layer-folders with a better name.

Worked example:
```text
# Bad — layer folders scatter checkout across the tree
lib/commerce/models/order.rb
lib/commerce/services/order_service.rb
lib/commerce/repositories/order_repository.rb

# Good — domain folder keeps checkout together
lib/commerce/checkout/order.rb
lib/commerce/checkout/order_service.rb
lib/commerce/checkout/order_repository.rb
```

**Enforcement:** review; a folder named for a technical role rather than a business domain is a design discussion in the PR.

## Cross-references

- `frozen_string_literal` header, 2-space indent, 100-column limit, and `rubocop-airbnb` baseline: [01-formatting-and-tooling.md](./01-formatting-and-tooling.md).
- File naming and `snake_case`/`CamelCase` alignment between filenames and constants: [02-naming-conventions.md](./02-naming-conventions.md).
- Sorbet `# typed: strict`, `sig` on every method, and `rbi` file placement in `sig/`: [03-type-safety-and-nil-discipline.md](./03-type-safety-and-nil-discipline.md).
- `Data.define` value objects and module-of-functions alternatives to class bags: [06-classes-and-data-modeling.md](./06-classes-and-data-modeling.md).
- Minimal public surface, keyword args, and `sig` on every public method — the promise the `Internal` namespace protects: [10-api-design.md](./10-api-design.md).
- Test directory structure mirroring `lib/`, and Minitest file conventions: [11-testing.md](./11-testing.md).
