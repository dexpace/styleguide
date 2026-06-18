---
name: csharp-aspnetcore-styleguide
description: Use when writing, editing, or reviewing ASP.NET Core code (minimal APIs, DI, EF Core, Kestrel) in a dexpace project — extends csharp-styleguide with ASP.NET Core rules (host/config, DI lifetimes, endpoints, persistence). Use alongside csharp-styleguide, not instead of it.
---

# ASP.NET Core styleguide

## When this applies

Editing or reviewing ASP.NET Core code. Triggering markers: `WebApplicationBuilder`, `Microsoft.AspNetCore.*` imports, a `DbContext`, `MapGet`/`MapPost` endpoints, `appsettings.json`. Priority: **correctness > performance > developer experience.**

## Inherited

First apply `csharp-styleguide`; this layer adds, and where stricter overrides, for ASP.NET Core.

## Language hard rules

These add to the core C# rules; they never weaken them. Where this layer is stricter, it wins for ASP.NET Core.

- Host: compose in a thin `Program.cs` (configure, register, build, pipeline, run); no business logic in it. Layer `IConfiguration` in fixed precedence; keep secrets out of source. Bind to strongly-typed `record` options with `ValidateOnStart`; never index raw `IConfiguration` in domain code. Gate on `IHostEnvironment`, never a hand-rolled flag; detailed errors in Development only.
- DI: constructor injection only — no property injection, no service-locator. Register the role interface, not the concrete type. State every lifetime deliberately; never let a Singleton capture a Scoped or Transient. Keep `ValidateScopes` and `ValidateOnBuild` on in every environment. Do not dispose injected dependencies; the container owns what it created.
- Endpoints: minimal APIs by default; reach for an MVC controller only when the surface needs the filter pipeline, and say why. Keep the handler thin — parse, dispatch to one injected service, map. Return `TypedResults`, never magic status integers. Bind `[AsParameters]` request records, not long parameter lists. Thread the request `CancellationToken` into every downstream call.
- Parse every boundary: route, query, body, headers, configuration, and query rows are untrusted until parsed into a validated domain `record` at the edge; reject invalid input with `ProblemDetails`. The DTO never reaches the core.
- EF Core: register `DbContext` Scoped (pool for throughput), never Singleton. `AsNoTracking` reads; track only to write. Disable lazy loading; load with explicit `Include` or project with `Select`. Never expose or serialize an entity — project to a DTO. One unit of work is one `SaveChangesAsync` in one transaction. Schema changes via checked-in migrations; never ungated `EnsureCreated`/auto-migrate in production. Dapper or `FromSql` only for measured hot paths, parameterized and parsed into a record.
- Serialization: `System.Text.Json` exclusively, no Newtonsoft; source generation on hot paths; configure `JsonSerializerOptions` once and reuse. Parse the deserialized DTO into a domain `record` at the edge; distinguish absent from null deliberately.
- Async request path: no blocking, no sync-over-async, `AllowSynchronousIO` off. No `.Result`/`.Wait()`. Move heavy CPU work off the request thread to a bounded background queue.
- Observability: `ILogger<T>` with named placeholders, never interpolated messages; `LoggerMessage` source-gen on hot paths; errors carry the exception object. OpenTelemetry traces and metrics via OTLP; carry the request correlation id from `Activity` on every log and span. Honest health: liveness checks the process, readiness the real dependencies. Redact secrets and PII before any sink; no `Console.WriteLine`.
- Performance and build: bounded caches with explicit expiry; `IHttpClientFactory` typed clients, never `new HttpClient()` per request; rate limiting plus Kestrel limits; stream large bodies. Containerize non-root on a minimal base with a locked multi-stage build; configuration and secrets from the environment, never the image. Graceful shutdown via `IHostApplicationLifetime` — handle `SIGTERM`, drain in flight, dispose, exit within a deadline.

## Before you finish — verify

```bash
dotnet format --verify-no-changes
dotnet build -warnaserror
dotnet test
```

## Full guide

- [README](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/README.md)
- [01 Host & Configuration](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/01-host-and-configuration.md)
- [02 Dependency Injection](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/02-dependency-injection.md)
- [03 Minimal APIs & Endpoints](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/03-minimal-apis-and-endpoints.md)
- [04 Persistence with EF Core](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/04-persistence-ef-core.md)
- [05 Serialization & Validation](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/05-serialization-and-validation.md)
- [06 Logging & Observability](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/06-logging-and-observability.md)
- [07 ASP.NET Performance](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/07-aspnet-performance.md)
- [08 Build & Deployment](https://github.com/dexpace/styleguide/blob/main/csharp-aspnetcore/08-build-and-deployment.md)

## Deep review

For a full audit (not a quick edit), read `reference/checklist.md` in this skill and walk every chapter.
