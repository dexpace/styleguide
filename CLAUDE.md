# CLAUDE.md

Pure-markdown styleguide canon for dexpace. No build, no tests — verification is mechanical (`grep`/`wc`) plus read-through.

## Architecture

- Root `README.md` defines 12 cross-cutting rules + the priority order (correctness > performance > DX). Per-language guides restate them natively and **extend a named canonical authority** (Google/PEPs/Kotlin conventions); where guidance conflicts, the authority wins, except deviations recorded in each guide's ledger.
- `kotlin/`, `python/`, `typescript/` share an aligned 15-chapter spine (same number → same topic; 03 = the language-safety slot). `go/` predates the spine — don't propagate its layout.
- Extension guides (`kotlin-jvm/`, `typescript-bun/`, `typescript-react/`) are **additive**: they never weaken a core rule; where stricter, they win for that runtime. Cross-reference core chapters, don't restate. The TS family is **bun-first** — `bun install`/`bun test`/`Bun.build` are the defaults, `tsc --noEmit` is the typecheck gate (Bun never type-checks), and React keeps Vitest+MSW for component tests as a recorded substitution; the retired Node guide lives at tag `node-guide-final`.

## Chapter format contract (typescript family; backport candidate)

- `# NN — Title` · scope paragraph · `## What good looks like` (one fenced exemplar + paragraph citing rule numbers) · `## Rules` with `### N.M — imperative title.` each carrying `**Reasoning, step by step:**` (numbered) + `**Enforcement:**` line, blank line after every `###` · `## Cross-references` last, relative links must resolve.
- Each guide README: authority chain, values, chapter index (exact filenames), rules restated natively, deviations ledger, and the verbatim closing `perfection over technical debt — debt never gets paid` exactly once.
- Verify gates: `grep -c '^### '` = chapter rule count · `grep -c '^## What good looks like'` = 1 · `grep -c 'Reasoning, step by step'` = rule count · code fences balanced (even count of triple-backtick lines).

## Hard rules the examples themselves must obey

- Examples must practice the guide's own rules — reviewers reject exemplars that break sibling chapters (no bare `as` without a why-comment, no `!`, no enums/parameter properties (`erasableSyntaxOnly`), discriminated unions kept whole so narrowing works).
- zod v4 top-level forms only: `z.email()` / `z.uuid()` / `z.url()`, never `z.string().email()` etc.
- House voice: imperative, dense, zero filler; never stack two em-dash clauses in one sentence.

## Git

- Conventional Commits (`type(scope): description`, imperative, ≤72-char subject); branches `type/short-description`; squash/rebase per `git-and-code-review.md`.
- **No `Co-Authored-By` trailers — commits author as the repo owner only.**
- Specs in `docs/superpowers/specs/`, plans in `docs/superpowers/plans/`.
