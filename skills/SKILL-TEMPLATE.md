# SKILL authoring contract

Every `<guide>-styleguide` skill follows this exact shape. The body is what loads into context when the skill fires, so keep it lean (~150–200 lines) and push the long tail to `reference/checklist.md`.

## Frontmatter

```yaml
---
name: <guide>-styleguide
description: Use when writing, editing, or reviewing <Language> (<exts>) source in a dexpace project — enforces the dexpace <Language> styleguide (<3–4 headline topics>, <N>-line function cap). Also use before committing <Language>, or when asked to review <Language> against the styleguide.
---
```

Only `name` and `description`. The `description` is the trigger — scope it to the language, its file extensions, and the dexpace context so it fires reliably without firing on unrelated work.

## Body sections (in this order)

1. `## When this applies` — file globs/context cues, plus the priority line `Correctness > performance > developer experience.`
2. `## Non-negotiables` — the 12 cross-cutting rules in one language-native line each. Source: the guide README's numbered rules list.
3. `## Language hard rules` — the reviewer-rejection gotchas, the function-size cap, and the assertion-density rule. These are the lines reviewers reject on.
4. `## Before you finish — verify` — the formatter, linter, and typecheck commands plus any `grep` self-checks, as runnable fenced commands.
5. `## Full guide` — the chapter index as `https://github.com/dexpace/styleguide/blob/main/<guide>/<chapter>.md` links.
6. `## Deep review` — one line: read `reference/checklist.md` for a full audit.

## `reference/checklist.md`

The exhaustive per-chapter rule list distilled from the guide's chapters, one `### NN — Chapter` heading per chapter with terse imperative bullets. Read on demand, not loaded on trigger.

## Verify gates (run before opening the PR)

- Frontmatter parses and has exactly `name` + `description`.
- Body has all six sections in order: `grep -n '^## ' SKILL.md`.
- Every `blob/main` URL resolves to a chapter file that exists in the repo.
- Function-size cap and assertion rule match the guide README.
- House voice: no line stacks two em-dash clauses; imperative; no filler.
- Code fences balanced (even count of triple-backtick lines), and examples obey the rules they teach.

## Extension skills

Additive only. Body opens: "First apply `<parent>-styleguide`; this layer adds, and where stricter overrides, for `<runtime>`." Then runtime-specific rules only. Never restate the core. Trigger on runtime markers, not just file extension.
