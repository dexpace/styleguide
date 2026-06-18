# Styleguide-enforcement skills — design

**Date:** 2026-06-18
**Status:** Approved for planning

## Goal

Ship Claude Code **skills** that make Claude reliably follow the dexpace styleguide
when writing, editing, or reviewing code in dexpace product repos — one skill per
guide, one PR per skill.

The styleguide repo is pure-markdown canon with no code to lint, so enforcement has
to happen where the code lives: the product repos. The skills carry the canon's hard
rules into those repos and into Claude's context at the moment it writes code.

## Scope

All ten guides get a skill:

- **Core (6):** `go`, `kotlin`, `python`, `typescript`, `csharp`, `ruby`.
- **Extensions (4):** `kotlin-jvm`, `typescript-bun`, `typescript-react`,
  `csharp-aspnetcore`.

Out of scope: the cross-cutting docs (`security.md`, `performance.md`,
`git-and-code-review.md`) as standalone skills — their rules are folded into each
language skill's non-negotiables where they bite.

## Architecture & distribution

This repo doubles as a **self-hosted Claude Code plugin**. Canon and skills ship from
one source of truth, so a skill's digest and the guide it distills can never drift
across repos.

```
styleguide/
  .claude-plugin/
    plugin.json                 # name: dexpace-styleguide
    marketplace.json            # /plugin marketplace add dexpace/styleguide
  skills/
    SKILL-TEMPLATE.md           # the anatomy contract (authoring reference)
    README.md                   # what these skills are, how to install
    go-styleguide/              SKILL.md + reference/checklist.md
    kotlin-styleguide/          SKILL.md + reference/checklist.md
    python-styleguide/          ...
    typescript-styleguide/
    csharp-styleguide/
    ruby-styleguide/
    kotlin-jvm-styleguide/      # extension -> cross-refs kotlin-styleguide
    typescript-bun-styleguide/  # extension -> cross-refs typescript-styleguide
    typescript-react-styleguide/# extension -> cross-refs typescript-styleguide
    csharp-aspnetcore-styleguide/# extension -> cross-refs csharp-styleguide
  kotlin/ python/ ...           # unchanged canon
```

A dexpace product repo installs once
(`/plugin marketplace add dexpace/styleguide`, then install the plugin). All ten
skills become available; each fires only on its own language via its trigger.

Chapter links use the canonical base
`https://github.com/dexpace/styleguide/blob/main/<guide>/<chapter>.md` so they resolve
from any repo, not just this one.

## Skill anatomy (the contract every SKILL.md follows)

Mirrors the repo's own chapter-format discipline: every skill has the same shape, so
they are mechanically verifiable and read consistently.

### Frontmatter

- `name`: `<guide>-styleguide` (e.g. `kotlin-styleguide`).
- `description`: the trigger — the only text in context until the skill fires, so it
  is tuned for reliable activation. Pattern:

  > Use when writing, editing, or reviewing Kotlin (`.kt`) source in a dexpace project
  > — enforces the dexpace Kotlin styleguide (nullability, sealed hierarchies,
  > assertion density, 60-line function cap). Also use before committing Kotlin, or
  > when asked to review Kotlin against the styleguide.

### Body (~150–200 lines, loaded on trigger)

1. `## When this applies` — file globs / context cues, plus the priority order
   **correctness > performance > developer experience**.
2. `## Non-negotiables` — the 12 cross-cutting README rules in language-native
   one-liners.
3. `## Language hard rules` — the reviewer-rejection gotchas, the function-size cap,
   and the assertion-density rule for this language. These are the lines reviewers
   actually reject on (e.g. Kotlin: no `!!`, no bare `as` without a why-comment,
   sealed unions kept whole so narrowing works, 60-line cap, ≥2 assertions/fn avg).
4. `## Before you finish — verify` — the mechanical gates as runnable commands:
   formatter, linter, typecheck, and `grep` self-checks the styleguide already uses.
5. `## Full guide` — the chapter index as `dexpace/styleguide` GitHub URLs.
6. `## Deep review` — instruction to read `reference/checklist.md` for a full
   audit rather than a quick edit.

### `reference/checklist.md` (read on demand)

The exhaustive per-chapter rule list distilled from the 15 (or, for extensions, 8)
chapters — used when doing a full review/audit. Kept out of the always-loaded body so
trigger-time context stays lean.

### Extension skills are additive

Body opens with: "First apply `<parent>-styleguide`; this layer adds, and where
stricter overrides, for `<runtime>`." Then only runtime-specific rules. Never restates
the core. Triggers on runtime markers (Spring/Ktor/JPA; Bun/`Bun.serve`; `.tsx`/React;
ASP.NET Core).

## Content sourcing

Each skill is **distilled from that guide's README + chapters**, never invented. The
digest stays faithful to canon; anything not inlined is one GitHub link away. Skills
obey house voice (imperative, dense, no double em-dash clauses) and the
examples-practice-their-own-rules discipline — any code fence in a skill must pass the
rules it teaches.

## PR plan (one PR per skill)

- **PR #0 — scaffold:** `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`,
  `skills/README.md`, `skills/SKILL-TEMPLATE.md`. No language content. Keeps each
  language PR purely about its language and gives clean history.
- **PR #1–6 — core skills**, in order: go, kotlin, python, typescript, csharp, ruby.
- **PR #7–10 — extension skills:** kotlin-jvm, typescript-bun, typescript-react,
  csharp-aspnetcore.

Each PR: branch `feat/<guide>-styleguide-skill` (scaffold:
`feat/styleguide-plugin-scaffold`), Conventional-Commit title, **no `Co-Authored-By`**,
self-contained description with no session/audit framing (per global PR rules).

## Per-skill verification (before each PR)

Mechanical, in the repo's spirit:

- Every chapter URL in the skill resolves on `dexpace/styleguide@main`.
- The digest's rule count, function-size cap, and assertion rule match the guide.
- `description` triggers cleanly on a representative prompt and file type.
- House-voice check: imperative, no double em-dash, no filler.
- Frontmatter parses; `reference/checklist.md` present and readable.

## Risks & mitigations

- **Digest drift from canon.** Mitigation: distill, don't invent; link every chapter;
  keep the digest small enough to diff against the guide by eye.
- **Over-broad triggers firing on unrelated work.** Mitigation: scope `description` to
  the language + file extension + dexpace context; extensions gate on runtime markers.
- **Context bloat.** Mitigation: hard rules inline, long tail in `reference/checklist.md`
  read only on demand.

## Out of scope / explicitly deferred

- Slash commands or agents (skills only for now).
- Auto-fix tooling or hooks that mutate code.
- A standalone cross-cutting `security`/`performance` skill.
