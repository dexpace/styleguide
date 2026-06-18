# Styleguide-Enforcement Skills Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship ten Claude Code skills (one per styleguide guide) that make Claude follow the dexpace styleguide when writing, editing, or reviewing code in product repos, packaged as a self-hosted plugin.

**Architecture:** This repo doubles as a Claude Code plugin. `.claude-plugin/` declares the plugin + a self-hosted marketplace. Each `skills/<guide>-styleguide/` holds a lean `SKILL.md` digest (loaded on trigger) plus `reference/checklist.md` (read on demand). Skills are distilled from the canon and link back to it by GitHub URL. One PR per skill (plus one scaffold PR).

**Tech Stack:** Markdown only. No build, no runtime. Verification is mechanical: `grep`/`wc`, frontmatter parse, URL-resolve, read-through.

## Global Constraints

- **Canonical URL base** for all chapter links: `https://github.com/dexpace/styleguide/blob/main/<guide>/<chapter>.md`.
- **Function-size caps** (verbatim): Go 70 · Kotlin 60 · Python 50 · TypeScript 70 · C# 70 · Ruby 25.
- **Assertion density:** ≥2 assertions per function on average, all languages (Python frames it as an explicit overlay, not native).
- **Priority order, verbatim:** correctness > performance > developer experience.
- **House voice:** imperative, dense, zero filler; never stack two em-dash clauses in one sentence.
- **Examples obey their own rules:** any code fence inside a skill must pass the rules that skill teaches (no `!!`, no bare `as`/`!`, no `enum`, zod v4 top-level forms `z.email()`/`z.uuid()`/`z.url()`, discriminated unions kept whole).
- **Git:** Conventional Commits, imperative, ≤72-char subject. Branch per skill `feat/<guide>-styleguide-skill`. **No `Co-Authored-By` trailers.** PR titles/descriptions self-contained, no session/audit/finding framing.
- **Skill frontmatter:** YAML with exactly `name` and `description`. `name` = `<guide>-styleguide`. No other keys.
- **SKILL.md body section order (fixed):** `## When this applies` · `## Non-negotiables` · `## Language hard rules` · `## Before you finish — verify` · `## Full guide` · `## Deep review`.

---

### Task 0: Plugin scaffold + marketplace + template

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `.claude-plugin/marketplace.json`
- Create: `skills/README.md`
- Create: `skills/SKILL-TEMPLATE.md`

**Interfaces:**
- Produces: the plugin manifest, the marketplace entry, and the authoring contract (`SKILL-TEMPLATE.md`) every later task copies. Later tasks rely on the section order and verify-gate list defined here.

- [ ] **Step 1: Write `.claude-plugin/plugin.json`**

```json
{
  "name": "dexpace-styleguide",
  "description": "Enforces the dexpace per-language styleguides (Go, Kotlin, Python, TypeScript, C#, Ruby and their runtime extensions) when writing, editing, or reviewing code.",
  "version": "0.1.0",
  "author": { "name": "dexpace" },
  "homepage": "https://github.com/dexpace/styleguide",
  "repository": "https://github.com/dexpace/styleguide",
  "license": "MIT"
}
```

- [ ] **Step 2: Write `.claude-plugin/marketplace.json`**

```json
{
  "name": "dexpace",
  "owner": { "name": "dexpace", "url": "https://github.com/dexpace" },
  "metadata": {
    "description": "dexpace engineering plugins",
    "version": "0.1.0"
  },
  "plugins": [
    {
      "name": "dexpace-styleguide",
      "source": "./",
      "description": "Per-language styleguide enforcement skills for dexpace projects."
    }
  ]
}
```

- [ ] **Step 3: Write `skills/SKILL-TEMPLATE.md`** — the authoring contract. Contents:

````markdown
# SKILL authoring contract

Every `<guide>-styleguide` skill follows this exact shape. The body is what
loads into context when the skill fires, so keep it lean (~150–200 lines) and
push the long tail to `reference/checklist.md`.

## Frontmatter

```yaml
---
name: <guide>-styleguide
description: Use when writing, editing, or reviewing <Language> (<exts>) source in a dexpace project — enforces the dexpace <Language> styleguide (<3–4 headline topics>, <N>-line function cap). Also use before committing <Language>, or when asked to review <Language> against the styleguide.
---
```

Only `name` and `description`. The `description` is the trigger — scope it to
the language, its file extensions, and the dexpace context so it fires reliably
without firing on unrelated work.

## Body sections (in this order)

1. `## When this applies` — file globs/context cues + the priority line
   `Correctness > performance > developer experience.`
2. `## Non-negotiables` — the 12 cross-cutting rules in one language-native
   line each. Source: the guide README's numbered rules list.
3. `## Language hard rules` — the reviewer-rejection gotchas, the function-size
   cap, and the assertion-density rule. These are the lines reviewers reject on.
4. `## Before you finish — verify` — the formatter/linter/typecheck commands and
   any `grep` self-checks, as runnable fenced commands.
5. `## Full guide` — the chapter index as
   `https://github.com/dexpace/styleguide/blob/main/<guide>/<chapter>.md` links.
6. `## Deep review` — one line: read `reference/checklist.md` for a full audit.

## `reference/checklist.md`

The exhaustive per-chapter rule list distilled from the guide's chapters, one
`### NN — Chapter` heading per chapter with terse imperative bullets. Read on
demand, not loaded on trigger.

## Verify gates (run before opening the PR)

- Frontmatter parses and has exactly `name` + `description`.
- Body has all six sections in order: `grep -n '^## ' SKILL.md`.
- Every `blob/main` URL resolves (chapter file exists in the repo).
- Function-size cap and assertion rule match the guide README.
- House voice: no line stacks two ` — ` em-dash clauses; imperative; no filler.
- Code fences balanced (even count of ``` lines) and examples obey the rules.

## Extension skills

Additive only. Body opens: "First apply `<parent>-styleguide`; this layer adds,
and where stricter overrides, for `<runtime>`." Then runtime-specific rules only.
Never restate the core. Trigger on runtime markers, not just file extension.
````

- [ ] **Step 4: Write `skills/README.md`**

```markdown
# dexpace styleguide skills

Claude Code skills that enforce the [dexpace styleguide](../README.md) when
writing, editing, or reviewing code. Each skill distills one guide into a lean
digest and links back to the canonical chapters.

## Install

```
/plugin marketplace add dexpace/styleguide
/plugin install dexpace-styleguide@dexpace
```

## Skills

| Skill | Language / runtime | Cap |
|---|---|---|
| `go-styleguide` | Go | 70 |
| `kotlin-styleguide` | Kotlin | 60 |
| `python-styleguide` | Python | 50 |
| `typescript-styleguide` | TypeScript | 70 |
| `csharp-styleguide` | C# | 70 |
| `ruby-styleguide` | Ruby | 25 |
| `kotlin-jvm-styleguide` | Kotlin on the JVM (extends `kotlin-styleguide`) | 60 |
| `typescript-bun-styleguide` | TypeScript on Bun (extends `typescript-styleguide`) | 70 |
| `typescript-react-styleguide` | React (extends `typescript-styleguide`) | 70 |
| `csharp-aspnetcore-styleguide` | ASP.NET Core (extends `csharp-styleguide`) | 70 |

Authoring contract: [`SKILL-TEMPLATE.md`](./SKILL-TEMPLATE.md).
```

- [ ] **Step 5: Verify scaffold**

Run: `cat .claude-plugin/plugin.json | python3 -m json.tool >/dev/null && cat .claude-plugin/marketplace.json | python3 -m json.tool >/dev/null && echo OK`
Expected: `OK` (both JSON files parse).

Run: `ls skills/SKILL-TEMPLATE.md skills/README.md`
Expected: both paths listed.

- [ ] **Step 6: Commit + PR**

```bash
git checkout -b feat/styleguide-plugin-scaffold
git add .claude-plugin skills/README.md skills/SKILL-TEMPLATE.md
git commit -m "feat: add styleguide plugin scaffold and skill template"
git push -u origin feat/styleguide-plugin-scaffold
gh pr create --title "feat: styleguide plugin scaffold and skill authoring contract" \
  --body "Adds the Claude Code plugin manifest, a self-hosted marketplace entry, and the SKILL authoring contract that the per-language styleguide skills follow. No language content yet — each language skill lands in its own PR."
```

---

### Task 1: go-styleguide skill (worked exemplar)

This task is fully written out; Tasks 2–10 follow the same shape with their own
facts. Read the go guide chapters before distilling so the digest stays faithful.

**Files:**
- Create: `skills/go-styleguide/SKILL.md`
- Create: `skills/go-styleguide/reference/checklist.md`

**Source chapters (read these, distill — do not invent):** `go/README.md` and `go/01`…`go/13`.

**Interfaces:**
- Consumes: `skills/SKILL-TEMPLATE.md` (the contract).
- Produces: the reference shape Tasks 2–10 copy.

- [ ] **Step 1: Write `skills/go-styleguide/SKILL.md`**

```markdown
---
name: go-styleguide
description: Use when writing, editing, or reviewing Go (.go) source in a dexpace project — enforces the dexpace Go styleguide (errors as values, structs + functions, bounded concurrency, 70-line function cap). Also use before committing Go, or when asked to review Go against the styleguide.
---

# Go styleguide

Extends the [Google Go Style Guide](https://google.github.io/styleguide/go/); where they conflict, Google wins, except the deviations below.

## When this applies

Editing `*.go`, or reviewing Go. Priority: **correctness > performance > developer experience.**

## Non-negotiables

1. Data + functions: structs for state, functions and small interfaces for behavior. No inheritance, no embedding for "is-a".
2. Explicit over implicit: every dependency in the signature, every error returned. Library options follow documented defaults.
3. Immutable by default: return new values; no mutating shared state; read-only intent at API edges.
4. Errors are values: handle every path. No `_ = err`. Wrap with `%w` plus context (cause, inputs, correlation ID).
5. Composition over inheritance: small consumer-defined interfaces composed together.
6. Transform, don't mutate: input in, new output out; state changes localized and visible.
7. Always say why: comments explain reasoning, not mechanics.
8. Assert aggressively: ≥2 assertions per function on average; preconditions, postconditions, invariants; split compound checks.
9. Limits on everything: bound every loop, channel, retry, buffer, timeout. No recursion in library code.
10. Small functions: one thing each; **70-line hard cap, aim 20–40**; blank lines between logical sections.
11. Performance from the outset: work with the runtime's grain; optimize slowest resource first (network > disk > memory > CPU).
12. Zero technical debt: do it right the first time.

## Language hard rules

- `%w` for wrapping, one level; place it at the end of the format string.
- No in-band errors (no sentinel `-1`/`""` for "not found") — return an explicit error or a typed result.
- Error strings lowercase, no trailing punctuation.
- `context.Context` is the first parameter, named `ctx`; never store it in a struct; no custom context types.
- Prefer synchronous APIs; the caller adds concurrency. Every goroutine has a known lifetime and a way to stop.
- Accept interfaces, return structs; interfaces defined by the consumer, kept small.
- `gofmt` and `golangci-lint` are law. `t.Fatal` only from the test goroutine.
- `crypto/rand` for secrets; never `math/rand`. `defer` for cleanup; timeouts on all external I/O.

## Before you finish — verify

```bash
gofmt -l .          # must print nothing
golangci-lint run   # must pass clean
go vet ./...
go test ./...
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/go/README.md)
- [01 Formatting & Tooling](https://github.com/dexpace/styleguide/blob/main/go/01-formatting-and-tooling.md)
- [02 Naming](https://github.com/dexpace/styleguide/blob/main/go/02-naming-conventions.md)
- [03 Error Handling](https://github.com/dexpace/styleguide/blob/main/go/03-error-handling.md)
- [04 Concurrency](https://github.com/dexpace/styleguide/blob/main/go/04-concurrency.md)
- [05 API Design](https://github.com/dexpace/styleguide/blob/main/go/05-api-design.md)
- [06 Testing](https://github.com/dexpace/styleguide/blob/main/go/06-testing.md)
- [07 Package Organization](https://github.com/dexpace/styleguide/blob/main/go/07-package-organization.md)
- [08 Logging](https://github.com/dexpace/styleguide/blob/main/go/08-logging.md)
- [09 Serialization](https://github.com/dexpace/styleguide/blob/main/go/09-serialization.md)
- [10 Resource Management](https://github.com/dexpace/styleguide/blob/main/go/10-resource-management.md)
- [11 Documentation](https://github.com/dexpace/styleguide/blob/main/go/11-documentation.md)
- [12 Variables & Declarations](https://github.com/dexpace/styleguide/blob/main/go/12-variables-and-declarations.md)
- [13 Performance Hints](https://github.com/dexpace/styleguide/blob/main/go/13-performance-hints.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
```

- [ ] **Step 2: Write `skills/go-styleguide/reference/checklist.md`**

Distill each `go/NN-*.md` chapter into one `### NN — Title` heading with terse imperative bullets covering that chapter's `### N.M` rules. Keep bullets specific (e.g. "03 — wrap with `%w` once, at the end; sentinels via `errors.Is`, typed errors via `errors.As`"). One heading per chapter 01–13.

- [ ] **Step 3: Verify the skill**

Run: `python3 -c "import sys,re; t=open('skills/go-styleguide/SKILL.md').read(); fm=t.split('---')[1]; assert 'name:' in fm and 'description:' in fm; print('frontmatter OK')"`
Expected: `frontmatter OK`

Run: `grep -n '^## ' skills/go-styleguide/SKILL.md`
Expected: the six sections, in order: When this applies, Non-negotiables, Language hard rules, Before you finish — verify, Full guide, Deep review.

Run: `grep -oE 'blob/main/go/[a-z0-9-]+\.md' skills/go-styleguide/SKILL.md | sed 's#blob/main/##' | sort -u | while read f; do test -f "$f" || echo "MISSING $f"; done; echo done`
Expected: `done` with no `MISSING` lines.

Run: `grep -c '70-line' skills/go-styleguide/SKILL.md`
Expected: `≥1` (cap present and correct).

Run: `awk '/```/{n++} END{print n%2}' skills/go-styleguide/SKILL.md`
Expected: `0` (fences balanced).

- [ ] **Step 4: House-voice + self-consistency read-through**

Read SKILL.md once. Confirm: no line stacks two ` — ` clauses; every example (if any) obeys the rules; bullets are imperative and filler-free.

- [ ] **Step 5: Commit + PR**

```bash
git checkout main && git pull && git checkout -b feat/go-styleguide-skill
git add skills/go-styleguide
git commit -m "feat: add go-styleguide enforcement skill"
git push -u origin feat/go-styleguide-skill
gh pr create --title "feat: Go styleguide enforcement skill" \
  --body "Adds a Claude Code skill that enforces the dexpace Go styleguide when writing, editing, or reviewing Go: the cross-cutting rules in Go-native terms, the reviewer-rejection gotchas (error wrapping, context rules, bounded goroutines, the 70-line cap), the gofmt/golangci-lint/vet/test gates, and links to the canonical chapters. A reference checklist supports full audits."
```

---

### Task 2: kotlin-styleguide skill

**Files:**
- Create: `skills/kotlin-styleguide/SKILL.md`
- Create: `skills/kotlin-styleguide/reference/checklist.md`

**Source chapters:** `kotlin/README.md` + `kotlin/01`…`kotlin/15`.

Follow Task 1's shape exactly. Facts for this skill:

- **Frontmatter description:** "Use when writing, editing, or reviewing Kotlin (.kt) source in a dexpace project — enforces the dexpace Kotlin styleguide (non-null by default, sealed result types, scope functions, 60-line function cap). Also use before committing Kotlin, or when asked to review Kotlin against the styleguide."
- **Authority line:** extends the Kotlin official coding conventions + Google Android Kotlin style guide; where they conflict, the official conventions win, except recorded deviations (100-column line limit; 60-line cap).
- **Cap:** 60-line hard cap, aim 15–30 (ktlint/detekt-enforced).
- **Language hard rules to inline:**
  - `!!` is banned. Resolve nullability at the boundary; internals take non-null types. Use Elvis with a real default or `error(...)`, `requireNotNull` for caller-fault contracts.
  - No bare `as`; prefer `as?` + handling, and any cast carries a why-comment.
  - Model absence/results as sealed hierarchies; `when` over a sealed type stays exhaustive (no `else` catch-all that defeats narrowing).
  - Errors: sealed `Result<T, E>` for expected failures; exceptions only for unrecoverable; no swallowing.
  - `by` delegation for reuse without "is-a"; data classes for state; no inheritance for code reuse.
  - Structured concurrency: coroutines scoped, dispatchers explicit, `Flow` cold, bounded `Channel`.
  - ktlint + detekt are law; 100-column lines; trailing commas; expression bodies where they read better.
- **Verify commands:** `ktlint --format` then `ktlint`; `detekt`; the project's Gradle build/test (`./gradlew detekt test` or `ktlint`/`detekt` directly).
- **Full guide:** link `kotlin/README.md` + chapters 01–15 (filenames from `ls kotlin/`).

- [ ] **Step 1:** Write `SKILL.md` per Task 1 shape with the facts above.
- [ ] **Step 2:** Write `reference/checklist.md`: one `### NN — Title` per chapter 01–15, terse bullets.
- [ ] **Step 3:** Verify (reuse Task 1 Step 3 commands, swapping `go`→`kotlin`; cap check `grep -c '60-line'`).
- [ ] **Step 4:** House-voice read-through.
- [ ] **Step 5:** Commit + PR.

```bash
git checkout main && git pull && git checkout -b feat/kotlin-styleguide-skill
git add skills/kotlin-styleguide
git commit -m "feat: add kotlin-styleguide enforcement skill"
git push -u origin feat/kotlin-styleguide-skill
gh pr create --title "feat: Kotlin styleguide enforcement skill" \
  --body "Adds a Claude Code skill enforcing the dexpace Kotlin styleguide: cross-cutting rules in Kotlin-native terms, the hard gotchas (no \`!!\`, no bare \`as\`, sealed result types kept exhaustive, the 60-line cap), the ktlint/detekt gates, and canonical chapter links, with a reference checklist for full audits."
```

---

### Task 3: python-styleguide skill

**Files:**
- Create: `skills/python-styleguide/SKILL.md`
- Create: `skills/python-styleguide/reference/checklist.md`

**Source chapters:** `python/README.md` + `python/01`…`python/15`.

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing Python (.py) source in a dexpace project — enforces the dexpace Python styleguide (full type hints, dataclasses + Protocols, structured asyncio, 50-line function cap). Also use before committing Python, or when asked to review Python against the styleguide."
- **Authority line:** extends PEP 8 / PEP 20 / PEP 484+604 / PEP 695 + Google Python Style Guide; where they conflict, the PEPs win. Python 3.12+.
- **Cap:** 50-line hard cap, aim 10–25 (tightest of the spine — bodies lack braces/types).
- **Language hard rules to inline:**
  - Type every public signature; `|` over `Optional`; `Protocol` for structural typing; no `Any` in public APIs. mypy strict is the gate.
  - `@dataclass(frozen=True, slots=True)` + `Protocol` over inheritance; no inheritance for reuse.
  - Errors: custom exception hierarchies, chain with `raise ... from`; no bare `except:`; `contextlib.suppress` for the deliberate ignore; fail fast.
  - Structured concurrency: `asyncio.TaskGroup` + `asyncio.timeout`; handle cancellation; `threading` only when forced.
  - Idioms: context managers, generators, comprehensions, EAFP, `pathlib`, f-strings, `match`/`case`.
  - **Assertion density is an explicit overlay** the language doesn't ask for: validate at every public boundary, split compound checks, fail fast. Say so plainly.
  - Resources: `with`/`async with`; `secrets`/`os.urandom` for secrets; timeouts on external I/O.
  - Ruff is lint + format; mypy strict.
- **Verify commands:** `ruff format .`; `ruff check .`; `mypy --strict .` (or project config); `pytest`.
- **Full guide:** `python/README.md` + chapters 01–15.

- [ ] **Step 1:** Write `SKILL.md`. **Step 2:** Write `reference/checklist.md` (15 chapters). **Step 3:** Verify (swap `python`; cap check `grep -c '50-line'`). **Step 4:** House-voice read. **Step 5:** Commit + PR:

```bash
git checkout main && git pull && git checkout -b feat/python-styleguide-skill
git add skills/python-styleguide
git commit -m "feat: add python-styleguide enforcement skill"
git push -u origin feat/python-styleguide-skill
gh pr create --title "feat: Python styleguide enforcement skill" \
  --body "Adds a Claude Code skill enforcing the dexpace Python styleguide: cross-cutting rules in Python-native terms, the hard rules (full type hints + mypy strict, frozen-slots dataclasses + Protocols over inheritance, structured asyncio with TaskGroup, the assertion-density overlay, the 50-line cap), the ruff/mypy/pytest gates, and canonical chapter links, with a reference checklist for full audits."
```

---

### Task 4: typescript-styleguide skill

**Files:**
- Create: `skills/typescript-styleguide/SKILL.md`
- Create: `skills/typescript-styleguide/reference/checklist.md`

**Source chapters:** `typescript/README.md` + `typescript/01`…`typescript/15`.

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing TypeScript (.ts) source in a dexpace project — enforces the dexpace TypeScript styleguide (no any/no enum, erasable syntax, Result unions, zod at boundaries, 70-line function cap). Also use before committing TypeScript, or when asked to review TypeScript against the styleguide."
- **Authority line:** extends Google TS Style Guide + ts.dev/style. TS ≥ 5.8, erasable syntax only (emits no runtime code), gts toolchain.
- **Cap:** 70-line hard cap (lint-enforced); `max-depth 3`, `max-params 3`.
- **Language hard rules to inline:**
  - `any` is banned — use `unknown` + narrowing. `as` needs a why-comment; prefer `satisfies`, guards, parse-don't-validate. Non-null `!` banned outside declared bridges.
  - `enum` and constructor parameter properties are banned (`erasableSyntaxOnly`) — use literal unions / `as const` maps.
  - Discriminated unions kept whole so narrowing works. Absence = `undefined`. `readonly` on public shapes. Branded primitives for identifiers.
  - No `||` for defaults — use `??` / `?.`. Pipelines (`map`/`filter`/`reduce`) over loops; name the steps.
  - Errors: `Error` subclasses, mandatory `cause` chaining, `catch (e: unknown)` + narrow, opt-in `Result` unions; programmer vs operational errors.
  - Concurrency: async/await only, `no-floating-promises`, `AbortSignal.timeout()`, bounded fan-out via a worker-pool helper.
  - zod **v4 top-level forms only**: `z.email()`, `z.uuid()`, `z.url()` — never `z.string().email()`. Types from `z.infer`. Validate at every boundary.
  - `const` default, `let` justified, `var` banned. kebab-case files. No `I` prefix, no `Async` suffix. gts is law; `tsc --noEmit` typechecks.
- **Verify commands:** `bun install`; `gts lint` (or `eslint`); `tsc --noEmit`; `bun test`.
- **Full guide:** `typescript/README.md` + chapters 01–15.

- [ ] **Step 1:** Write `SKILL.md`. **Step 2:** Write `reference/checklist.md` (15 chapters). **Step 3:** Verify (swap `typescript`; cap check `grep -c '70-line'`; confirm any zod fence uses top-level forms). **Step 4:** House-voice read. **Step 5:** Commit + PR:

```bash
git checkout main && git pull && git checkout -b feat/typescript-styleguide-skill
git add skills/typescript-styleguide
git commit -m "feat: add typescript-styleguide enforcement skill"
git push -u origin feat/typescript-styleguide-skill
gh pr create --title "feat: TypeScript styleguide enforcement skill" \
  --body "Adds a Claude Code skill enforcing the dexpace TypeScript styleguide: cross-cutting rules in TS-native terms, the hard rules (no any/no enum, erasable syntax only, no bare as or non-null !, discriminated unions kept whole, ?? over ||, zod v4 top-level forms at boundaries, the 70-line cap), the gts/tsc/bun-test gates, and canonical chapter links, with a reference checklist for full audits."
```

---

### Task 5: csharp-styleguide skill

**Files:**
- Create: `skills/csharp-styleguide/SKILL.md`
- Create: `skills/csharp-styleguide/reference/checklist.md`

**Source chapters:** `csharp/README.md` + `csharp/01`…`csharp/15`.

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing C# (.cs) source in a dexpace project — enforces the dexpace C# styleguide (NRT as law, records + pattern matching, async all the way, 70-line method cap). Also use before committing C#, or when asked to review C# against the styleguide."
- **Authority line:** extends the .NET Runtime C# Coding Style. C# 14 / .NET 10. Deviations: no `I` prefix on first-party interfaces, no `Async` suffix, stricter null-forgiving ban, 70-line cap.
- **Cap:** 70-line hard cap (analyzer-enforced); nesting two, three at most.
- **Language hard rules to inline:**
  - NRT is law: `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>`. Null-forgiving `!` banned outside declared bridges / `[MemberNotNull]` proofs. No `#nullable disable`. `dynamic` banned. `default!` only at proven boundaries.
  - Data + functions: immutable `record` DTOs, `required` + `init`, struct vs class vs record chosen deliberately; no inheritance for reuse.
  - Errors: specific exception types; no bare `catch (Exception)` without a filter; `throw;` to rethrow + inner-exception chaining; `ArgumentNullException.ThrowIf*` guards; opt-in `Result` for expected failures; no control-flow exceptions.
  - Concurrency: async/await throughout; `async void` banned; `CancellationToken` + timeout everywhere; `ConfigureAwait(false)` in libraries; never `.Result`/`.Wait()`; `Channel<T>`, bounded parallelism.
  - Naming: PascalCase types/members, `_camelCase` fields, no `I` prefix on first-party interfaces (BCL interfaces keep theirs), no `Async` suffix, language keywords over BCL types, `nameof`.
  - Idioms: `switch` expressions, LINQ pipelines, collection expressions `[..]`, `is not null`, `??`/`?.`, raw strings, `nameof`.
  - `dotnet format` + `.editorconfig`; Allman braces, four spaces, file-scoped namespaces; `<AnalysisLevel>`.
- **Verify commands:** `dotnet format --verify-no-changes`; `dotnet build -warnaserror`; `dotnet test`.
- **Full guide:** `csharp/README.md` + chapters 01–15.

- [ ] **Step 1:** Write `SKILL.md`. **Step 2:** Write `reference/checklist.md` (15 chapters). **Step 3:** Verify (swap `csharp`; cap check `grep -c '70-line'`). **Step 4:** House-voice read. **Step 5:** Commit + PR:

```bash
git checkout main && git pull && git checkout -b feat/csharp-styleguide-skill
git add skills/csharp-styleguide
git commit -m "feat: add csharp-styleguide enforcement skill"
git push -u origin feat/csharp-styleguide-skill
gh pr create --title "feat: C# styleguide enforcement skill" \
  --body "Adds a Claude Code skill enforcing the dexpace C# styleguide: cross-cutting rules in C#-native terms, the hard rules (NRT as law with the null-forgiving ban, immutable records + required/init, async throughout with no async void, no first-party I prefix or Async suffix, the 70-line cap), the dotnet format/build-warnaserror/test gates, and canonical chapter links, with a reference checklist for full audits."
```

---

### Task 6: ruby-styleguide skill

**Files:**
- Create: `skills/ruby-styleguide/SKILL.md`
- Create: `skills/ruby-styleguide/reference/checklist.md`

**Source chapters:** `ruby/README.md` + `ruby/01`…`ruby/15`.

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing Ruby (.rb) source in a dexpace project — enforces the dexpace Ruby styleguide (Sorbet # typed: strict, Data.define value objects, frozen by default, 25-line method cap). Also use before committing Ruby, or when asked to review Ruby against the styleguide."
- **Authority line:** extends Airbnb + Shopify + rubystyle.guide. Ruby 4.0+. Deviation/addition: mandatory Sorbet `# typed: strict` with runtime `sig` checking; `find`/`select`/`reduce`/`map` naming.
- **Cap:** 25-line hard cap, aim 5–15 (cop-enforced); nesting three, params four.
- **Language hard rules to inline:**
  - `# typed: strict`; a `sig` on every method; `T.let`/`T.cast` need a why-comment; `T.must` banned outside bridges. `&.`, `fetch` over `[]`, no `nil` across boundaries, parse don't validate.
  - Frozen by default; `Data.define` value objects; data + functions — modules of functions over class-method bags; composition via mixins; `T::Enum`/sealed; make illegal states unrepresentable.
  - Errors: subclass `StandardError`; never `rescue Exception`; no flow-control by exception; implicit `begin`; no empty/`nil`/modifier rescue; raise class + message; `cause` chaining.
  - Naming: `snake_case`/`CamelCase`/`SCREAMING_SNAKE_CASE`; predicate `?`; bang `!` only when paired with a non-bang; no `is_`/`get_`; one class per file; no magic numbers.
  - Idioms: Enumerable pipelines, `&:sym`, `tap`/`then`, `case/in`, no `for`, `{}` vs `do..end` by use, no monkey-patching, squiggly heredocs, `Time` over `DateTime`.
  - Concurrency: `Mutex` + immutable sharing, Ractors for parallelism, Fibers + `async`, bounded pools/queues, deadline timeouts over `Timeout.timeout`.
  - `rubocop-airbnb`; `srb tc` is the first suite; Minitest.
- **Verify commands:** `srb tc`; `rubocop` (or `rubocop -a` to autocorrect); `bundle exec rake test` (Minitest).
- **Full guide:** `ruby/README.md` + chapters 01–15.

- [ ] **Step 1:** Write `SKILL.md`. **Step 2:** Write `reference/checklist.md` (15 chapters). **Step 3:** Verify (swap `ruby`; cap check `grep -c '25-line'`). **Step 4:** House-voice read. **Step 5:** Commit + PR:

```bash
git checkout main && git pull && git checkout -b feat/ruby-styleguide-skill
git add skills/ruby-styleguide
git commit -m "feat: add ruby-styleguide enforcement skill"
git push -u origin feat/ruby-styleguide-skill
gh pr create --title "feat: Ruby styleguide enforcement skill" \
  --body "Adds a Claude Code skill enforcing the dexpace Ruby styleguide: cross-cutting rules in Ruby-native terms, the hard rules (Sorbet # typed: strict with a sig on every method and T.must banned, frozen Data.define value objects, never rescue Exception, the 25-line cap), the srb tc/rubocop/minitest gates, and canonical chapter links, with a reference checklist for full audits."
```

---

### Task 7: kotlin-jvm-styleguide skill (extension)

**Files:**
- Create: `skills/kotlin-jvm-styleguide/SKILL.md`
- Create: `skills/kotlin-jvm-styleguide/reference/checklist.md`

**Source chapters:** `kotlin-jvm/README.md` + `kotlin-jvm/01`…`kotlin-jvm/08`.

Extension shape — body opens: "First apply `kotlin-styleguide`; this layer adds, and where stricter overrides, for Kotlin on the JVM." Then JVM-only rules. Six body sections still apply, but `## Non-negotiables` is replaced by `## Inherited` (one line pointing to `kotlin-styleguide`), and `## Language hard rules` becomes the JVM additions.

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing Kotlin that runs on the JVM (Spring, Ktor, JPA, Gradle) in a dexpace project — extends kotlin-styleguide with JVM-specific rules (Java interop, virtual threads vs coroutines, framework wiring, persistence). Use alongside kotlin-styleguide, not instead of it."
- **Runtime markers (triggering cues, state in `## When this applies`):** Spring/Ktor imports, `@Configuration`/`@Entity`/`@Transactional`, `build.gradle.kts`, SLF4J.
- **JVM hard rules to inline (from chapters):**
  - Interop: `@Jvm*` annotations where Java callers need them; treat platform types as nullable; file-level null annotations.
  - Concurrency: virtual threads (Loom) vs coroutines chosen deliberately; Reactor only at boundaries; `CompletableFuture` bridges; MDC propagated across coroutines.
  - Frameworks: constructor injection only (no field injection); `@Configuration` over `@Component`; `@ConfigurationProperties`; explicit transactional boundaries.
  - Persistence: JPA with `kotlin-jpa`; never expose entities — map to data classes at the edge; `LAZY` defaults; equality by business key.
  - Serialization: kotlinx.serialization vs Jackson chosen deliberately; time types explicit; null-vs-absent JSON semantics.
  - Logging: SLF4J + kotlin-logging; lazy messages; MDC correlation; PII masking; no `println`.
  - Build: Gradle Kotlin DSL, version catalogs, toolchains, `binary-compatibility-validator`.
- **Verify commands:** `./gradlew ktlintCheck detekt`; `./gradlew build`; `./gradlew test`.
- **Full guide:** `kotlin-jvm/README.md` + chapters 01–08.

- [ ] **Step 1:** Write `SKILL.md` (extension shape). **Step 2:** Write `reference/checklist.md` (8 chapters). **Step 3:** Verify (swap `kotlin-jvm`, 8 chapters; confirm it references `kotlin-styleguide` and adds no weaker rule). **Step 4:** House-voice read. **Step 5:** Commit + PR:

```bash
git checkout main && git pull && git checkout -b feat/kotlin-jvm-styleguide-skill
git add skills/kotlin-jvm-styleguide
git commit -m "feat: add kotlin-jvm-styleguide enforcement skill"
git push -u origin feat/kotlin-jvm-styleguide-skill
gh pr create --title "feat: Kotlin-on-JVM styleguide enforcement skill" \
  --body "Adds an additive Claude Code skill for Kotlin on the JVM that layers on top of kotlin-styleguide: Java interop annotations, virtual-threads-vs-coroutines guidance, constructor-injection framework wiring, JPA entity mapping, SLF4J logging, and Gradle build rules, plus canonical chapter links. It never weakens a core rule; where stricter it wins for the JVM."
```

---

### Task 8: typescript-bun-styleguide skill (extension)

**Files:**
- Create: `skills/typescript-bun-styleguide/SKILL.md`
- Create: `skills/typescript-bun-styleguide/reference/checklist.md`

**Source chapters:** `typescript-bun/README.md` + `typescript-bun/01`…`typescript-bun/08`.

Extension shape (opens: "First apply `typescript-styleguide`; this layer adds, and where stricter overrides, for TypeScript on Bun.").

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing TypeScript that runs on Bun (Bun.serve, Hono, Bun.SQL, bun test) in a dexpace project — extends typescript-styleguide with Bun-specific rules (runtime pinning, the tsc typecheck gate, HTTP services, persistence). Use alongside typescript-styleguide, not instead of it."
- **Runtime markers:** `bun.lock`, `bunfig.toml`, `Bun.serve`, `bun:sqlite`, `Bun.SQL`, Hono imports.
- **Bun hard rules to inline:**
  - `bun install` + committed `bun.lock`; exact version pin (`.bun-version`); ESM-only; one process per container. **Bun never type-checks — `tsc --noEmit` is the gate.** `moduleResolution: "bundler"`.
  - No blocking in request paths; no `*Sync` calls; measure event-loop lag; bounded Bun Workers for CPU; Web Streams with backpressure.
  - Hono on `Bun.serve`; thin handlers; zod validator middleware; centralized error mapping; correlation IDs; timeouts; `app.request()` injection testing.
  - `Bun.SQL` (Postgres) with Drizzle; bounded pools; parameterized tagged-template queries; `bun:sqlite` for embedded; rows parsed at the edge.
  - zod at every boundary; types from `z.infer`; never trust `JSON.parse`; `Bun.env` parsed once.
  - Structured JSON logging (pino on Bun); correlation via `AsyncLocalStorage`; no `console.log`.
  - `Bun.build` for bundling; `bun build --compile` for single-file executables; publint + attw gate.
- **Verify commands:** `bun install --frozen-lockfile`; `tsc --noEmit`; `bun test`.
- **Full guide:** `typescript-bun/README.md` + chapters 01–08.

- [ ] **Step 1–5:** As Task 7, swapping `typescript-bun` (8 chapters), parent `typescript-styleguide`. PR:

```bash
git checkout main && git pull && git checkout -b feat/typescript-bun-styleguide-skill
git add skills/typescript-bun-styleguide
git commit -m "feat: add typescript-bun-styleguide enforcement skill"
git push -u origin feat/typescript-bun-styleguide-skill
gh pr create --title "feat: TypeScript-on-Bun styleguide enforcement skill" \
  --body "Adds an additive Claude Code skill for TypeScript on Bun that layers on top of typescript-styleguide: pinned Bun runtime with a committed lockfile, the tsc --noEmit typecheck gate (Bun never type-checks), Hono on Bun.serve with zod validation, Bun.SQL + Drizzle persistence, structured logging, and Bun.build distribution, plus canonical chapter links. It never weakens a core rule; where stricter it wins for Bun."
```

---

### Task 9: typescript-react-styleguide skill (extension)

**Files:**
- Create: `skills/typescript-react-styleguide/SKILL.md`
- Create: `skills/typescript-react-styleguide/reference/checklist.md`

**Source chapters:** `typescript-react/README.md` + `typescript-react/01`…`typescript-react/08`.

Extension shape (opens: "First apply `typescript-styleguide`; this layer adds, and where stricter overrides, for React.").

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing React components (.tsx) in a dexpace project — extends typescript-styleguide with React rules (function components, hooks discipline, server state in TanStack Query, accessibility). Use alongside typescript-styleguide, not instead of it."
- **Runtime markers:** `.tsx` files, `react`/`react-dom` imports, JSX, hooks.
- **React hard rules to inline (lead with modern-first React):**
  - Function components only; props typed as interfaces; no `React.FC`; `ref` is a prop (no `forwardRef`); one component per file.
  - Rules of Hooks; custom hooks are the unit of reuse; dependency arrays correct; `useRef` vs `useState` chosen deliberately; stable identities.
  - Server state in TanStack Query; client state in `useState`/`useReducer`; `<Context value>` wires dependencies, not data; per-domain Zustand for global dynamic state; no god-store; no Redux.
  - Data/forms: queries + mutations with cache invalidation; optimistic updates via `useOptimistic`; form Actions + `useActionState`; react-hook-form + zod for complex validation; validate at the edge.
  - Structure: feature folders; route-level code splitting; colocated UI; no barrels below the package root.
  - The React compiler does memoization — don't hand-memoize; profile before tuning.
  - Accessibility: semantic HTML first; ARIA only to fill gaps; keyboard reachable; focus managed; `eslint-plugin-jsx-a11y`.
  - **Testing substitution (record it):** React keeps Vitest + MSW (+ Testing Library, user-event, role-based queries) for component tests, not bun test.
- **Verify commands:** `gts lint` (or eslint + jsx-a11y); `tsc --noEmit`; `vitest run`.
- **Full guide:** `typescript-react/README.md` + chapters 01–08.

- [ ] **Step 1–5:** As Task 7, swapping `typescript-react` (8 chapters), parent `typescript-styleguide`. PR:

```bash
git checkout main && git pull && git checkout -b feat/typescript-react-styleguide-skill
git add skills/typescript-react-styleguide
git commit -m "feat: add typescript-react-styleguide enforcement skill"
git push -u origin feat/typescript-react-styleguide-skill
gh pr create --title "feat: React styleguide enforcement skill" \
  --body "Adds an additive Claude Code skill for React that layers on top of typescript-styleguide: function components with ref-as-prop and no React.FC, hooks discipline, server state in TanStack Query with client state local, useOptimistic and form Actions, accessibility-first markup, and the compiler-does-memoization stance, plus the recorded Vitest+MSW test substitution and canonical chapter links. It never weakens a core rule; where stricter it wins for React."
```

---

### Task 10: csharp-aspnetcore-styleguide skill (extension)

**Files:**
- Create: `skills/csharp-aspnetcore-styleguide/SKILL.md`
- Create: `skills/csharp-aspnetcore-styleguide/reference/checklist.md`

**Source chapters:** `csharp-aspnetcore/README.md` + `csharp-aspnetcore/01`…`csharp-aspnetcore/08`.

Extension shape (opens: "First apply `csharp-styleguide`; this layer adds, and where stricter overrides, for ASP.NET Core.").

Facts:

- **Frontmatter description:** "Use when writing, editing, or reviewing ASP.NET Core code (minimal APIs, DI, EF Core, Kestrel) in a dexpace project — extends csharp-styleguide with ASP.NET Core rules (host/config, DI lifetimes, endpoints, persistence). Use alongside csharp-styleguide, not instead of it."
- **Runtime markers:** `WebApplicationBuilder`, `Microsoft.AspNetCore.*`, `DbContext`, `MapGet`/`MapPost`, `appsettings.json`.
- **ASP.NET Core hard rules to inline:**
  - `WebApplicationBuilder`; layered `IConfiguration`; the Options pattern with `ValidateOnStart`; secrets out of source.
  - Constructor injection; explicit lifetimes; no captive dependencies; register the interface; no service-locator; `ValidateScopes`/`ValidateOnBuild`.
  - Minimal APIs over controllers; route groups; `TypedResults`; endpoint filters for cross-cutting; parse at the boundary; `ProblemDetails`; versioning; OpenAPI.
  - EF Core: scoped `DbContext` (or pool); `AsNoTracking` reads; no lazy loading; explicit `Include`; project to DTOs; never serialize entities; migrations; transactions as units; Dapper for hot paths.
  - Serialization: `System.Text.Json` + source generation; no Newtonsoft; options configured once; parse into records at the edge.
  - Observability: `ILogger<T>` structured logging; `LoggerMessage` source-gen on hot paths; OpenTelemetry; health checks; correlation via `Activity`; PII redaction; no `Console.WriteLine`.
  - Performance/build: Kestrel limits; output caching + `HybridCache`; `IHttpClientFactory` pooling; rate limiting; async throughout; chiseled/distroless containers; graceful shutdown via `IHostApplicationLifetime`.
- **Verify commands:** `dotnet format --verify-no-changes`; `dotnet build -warnaserror`; `dotnet test`.
- **Full guide:** `csharp-aspnetcore/README.md` + chapters 01–08.

- [ ] **Step 1–5:** As Task 7, swapping `csharp-aspnetcore` (8 chapters), parent `csharp-styleguide`. PR:

```bash
git checkout main && git pull && git checkout -b feat/csharp-aspnetcore-styleguide-skill
git add skills/csharp-aspnetcore-styleguide
git commit -m "feat: add csharp-aspnetcore-styleguide enforcement skill"
git push -u origin feat/csharp-aspnetcore-styleguide-skill
gh pr create --title "feat: ASP.NET Core styleguide enforcement skill" \
  --body "Adds an additive Claude Code skill for ASP.NET Core that layers on top of csharp-styleguide: WebApplicationBuilder host/config with the validated Options pattern, constructor DI with explicit lifetimes, minimal APIs with TypedResults and ProblemDetails, EF Core with AsNoTracking and DTO projection, System.Text.Json source generation, and structured observability, plus canonical chapter links. It never weakens a core rule; where stricter it wins for ASP.NET Core."
```

---

## Self-review notes

- **Spec coverage:** distribution (Task 0), all 6 core skills (Tasks 1–6), all 4 extensions (Tasks 7–10), skill anatomy (SKILL-TEMPLATE in Task 0, applied in Task 1), per-skill verification (Step 3/4 in each task), one PR per skill (each task ends in its own PR). Marketplace included (Task 0 Step 2). ✓
- **Chapter filenames:** Tasks reference chapters by number; pull exact filenames from `ls <guide>/` when writing the `## Full guide` links (Task 1 lists Go's verbatim as the model).
- **Cap consistency:** Go 70, Kotlin 60, Python 50, TS 70, C# 70, Ruby 25, JVM/Bun/React/ASP.NET inherit their parent's cap.
