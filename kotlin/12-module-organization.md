# 12 — Module Organization

How code is grouped, named, and exposed across boundaries. The shape of your module graph constrains every refactor.

## Rules

### 12.1 — Group by feature, not by technical layer.

**Reasoning, step by step:**
1. `com.acme.checkout.api`, `com.acme.checkout.domain`, `com.acme.checkout.storage` keeps a feature's code together.
2. `com.acme.controllers`, `com.acme.services`, `com.acme.repositories` scatters each feature across all three layers — every feature change touches three packages, every layer change touches every feature.
3. Feature-shaped layout localizes change: a new checkout requirement modifies the `checkout` package and nothing else.
4. Cross-feature shared code (DTOs, primitive types, errors) lives in a sibling `common` or `shared` package. Keep this minimal — every entry pulls every feature toward it.

### 12.2 — Mirror package structure in the source tree.

**Reasoning, step by step:**
1. Kotlin doesn't *require* source layout to match package, but every tool (IDE, refactoring, code-search) assumes it.
2. `src/main/kotlin/com/acme/checkout/api/CheckoutEndpoint.kt` is the path for `package com.acme.checkout.api`.
3. Don't flatten or nest weirdly. The cost of getting it wrong is invisible-until-it-isn't.

### 12.3 — One module per deployable boundary, plus narrow shared modules.

**Reasoning, step by step:**
1. A module is the unit of separate compilation and `internal` visibility scope.
2. Default to a small number of modules: one per service / deployable, plus narrow `*-api` or `*-shared` modules where it pays.
3. The cost of a module is build-time and cognitive: more modules = slower builds, harder navigation. The benefit is enforced boundaries.
4. **Decision rule:** new module when (a) the code has an independent lifecycle (different release cadence), (b) the boundary is genuinely public (a client library, an SPI), or (c) the team is large enough that ownership benefits from compile-time separation.

### 12.4 — `internal` is the workhorse visibility for cross-package, same-module code.

**Reasoning, step by step:**
1. `private` (file/class) is too narrow for code shared across packages within a module.
2. `public` is too wide — it exposes the symbol outside the module forever.
3. `internal` is exactly right: visible to the whole module, invisible outside.
4. Reach for `internal` aggressively. Only widen to `public` when an external caller actually needs the symbol.

### 12.5 — No cyclic dependencies. Ever.

**Reasoning, step by step:**
1. Module A depends on B. B must not depend on A — directly or transitively.
2. Cycles make incremental builds slow and ABI changes invisible.
3. CI must check for cycles (Gradle's module graph + an analyzer task, or `jdeps` on JVM).
4. **Resolving cycles:** extract the shared abstraction into a third module both depend on. Most cycles arise from a "shared utility" that's actually two unrelated utilities glued together.

### 12.6 — Public API surface is a deliberate, written-down list.

**Reasoning, step by step:**
1. The set of `public` symbols in a module *is* its API.
2. Maintain a snapshot (`api/*.api` files with `binary-compatibility-validator` on JVM, see [JVM guide ch. 08](../kotlin-jvm/08-build-and-distribution.md)) so changes are visible in diffs.
3. A PR that changes the snapshot file changes the API. Review accordingly.
4. This rule applies even to internal-use modules — the snapshot catches accidental over-exposure.

### 12.7 — Generated code lives in its own source set and isn't edited.

**Reasoning, step by step:**
1. Generated code (Protobuf, OpenAPI, Apollo, kotlinx.serialization helpers) goes in `src/main/generated/kotlin` or the build's `generated-src` output.
2. It's never hand-edited. If you need to change generation, change the generator config.
3. The generator runs as a build step; check the output into source control only if the generation is slow and rare. Otherwise generate on every build.

### 12.8 — Documentation lives with the code: README per module, KDoc per public symbol.

**Reasoning, step by step:**
1. A module has a `README.md` at its root explaining (a) what it does, (b) who uses it, (c) the public entry points.
2. Every `public` symbol has KDoc. Every `internal` symbol has KDoc when the name doesn't fully document.
3. **Anti-pattern:** READMEs that explain "what is Kotlin" or "how to run Gradle." Link out to canonical sources.

### 12.9 — Test code organization mirrors production.

**Reasoning, step by step:**
1. `src/test/kotlin/com/acme/checkout/CheckoutEndpointTest.kt` tests `src/main/kotlin/com/acme/checkout/CheckoutEndpointTest.kt`.
2. Same package as the production code → tests can call `internal` symbols without ceremony.
3. Test utilities (`UserFixture`, `anyOrder()`) live in `src/test/kotlin` under the package they're closest to, or in a `*-test-fixtures` source set if they're shared across modules.
4. Integration tests in a separate source set (`src/integrationTest`) if you have any — keep them out of the main test run unless they're fast.

### 12.10 — `expect`/`actual` declarations for multiplatform — and only where needed.

**Reasoning, step by step:**
1. Kotlin Multiplatform lets common code declare `expect` and per-platform `actual` implementations.
2. Use `expect`/`actual` only when the platform abstraction *is the API*. Most code should be platform-agnostic in `commonMain`.
3. For server-side JVM-only projects, this is irrelevant — and that's fine.
4. **Note:** this rule is here for projects that may go multiplatform. JVM-only readers can skip.

## Cross-references

- Visibility modifiers and API design: chapter 10.
- Public API stability checks (`binary-compatibility-validator`) on JVM: [JVM guide ch. 08](../kotlin-jvm/08-build-and-distribution.md).
- Package naming: chapter 02.
