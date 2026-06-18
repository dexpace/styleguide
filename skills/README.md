# dexpace styleguide skills

Claude Code skills that enforce the [dexpace styleguide](../README.md) when writing, editing, or reviewing code. Each skill distills one guide into a lean digest and links back to the canonical chapters.

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
