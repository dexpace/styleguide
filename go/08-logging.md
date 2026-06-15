# 08 — Logging

Logs are structured data, not prose. Use `log/slog` for leveled, key-value output that a machine can index and a human can read. Pass the logger as a dependency, never reach for the global default in library code. Mask secrets and PII at the type level so they cannot leak however a value is logged, guard expensive log arguments behind a level check, and log a failure once — at the boundary where you stop returning it — not at every layer it passes through.

## What good looks like

```go
// Token redacts itself however it is logged.
type Token struct {
    Value     string
    ExpiresAt time.Time
}

func (t Token) LogValue() slog.Value {
    return slog.StringValue("Token(***)")
}

// Application configures the logger once, at startup.
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))

// Library code accepts the logger as a dependency.
type Client struct {
    logger *slog.Logger
    // ...
}

func NewClient(logger *slog.Logger, opts ...Option) *Client {
    return &Client{logger: logger}
}

func (c *Client) Handle(ctx context.Context, r *Request) {
    // Context flows through the request via With; not reconstructed per call.
    logger := c.logger.With("request_id", r.ID, "user_id", r.UserID)

    start := time.Now()
    status, err := c.process(ctx, r)
    elapsed := time.Since(start)

    if err != nil {
        logger.Error("request failed", "error", err, "endpoint", r.Endpoint)
        return // boundary: log is the final action for this failure
    }

    logger.Info("request completed",
        "method", r.Method, "path", r.Path, "status", status,
        "duration_ms", elapsed.Milliseconds())

    // Expensive serialization runs only when DEBUG is enabled.
    if logger.Enabled(ctx, slog.LevelDebug) {
        logger.DebugContext(ctx, "state snapshot", "state", expensiveDump(r))
    }
}
```

This module logs through `log/slog` (8.1) with the logger injected as a dependency (8.2). Every record uses key-value attributes, never `fmt.Sprintf` (8.3); levels are chosen by meaning (8.4); the `Token`'s `LogValue` redacts the secret automatically (8.5, 8.11); request context is attached once with `With` (8.6); the error rides as a structured attribute (8.7); the expensive dump is guarded by a level check (8.8); and the failure is logged at the boundary instead of being logged-and-returned (8.9).

## Rules

### 8.1 — Log through `log/slog`, not a third-party logger.

**Reasoning, step by step:**
1. `log/slog` (Go 1.21+) is structured, leveled, and in the standard library — no dependency, no version drift, one logging model the whole codebase shares.
2. Reach for a third-party logger only when slog genuinely lacks the capability you need (e.g., log rotation), not by default.

**Enforcement:** review; `golangci-lint` (`depguard`) to block disallowed logging imports.

### 8.2 — Inject the logger; never use the global default in library code.

**Reasoning, step by step:**
1. Pass `*slog.Logger` as a parameter or struct field. A library that reaches for the global default logger hides a dependency and cannot be configured or tested by its caller.
2. Applications own configuration: they construct the logger once at startup and hand it down.

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
```

3. Library code accepts the logger as a dependency rather than constructing or defaulting one:

```go
type Client struct {
    logger *slog.Logger
    // ...
}

func NewClient(logger *slog.Logger, opts ...Option) *Client {
    return &Client{logger: logger}
}
```

**Enforcement:** review; `golangci-lint` (`sloglint` no-global) to flag global-default usage.

### 8.3 — Log key-value attributes, never `fmt.Sprintf`.

**Reasoning, step by step:**
1. Structured attributes are indexable: a log aggregator can filter on `status` or `path` without parsing prose. A formatted string is opaque to every tool downstream.
2. Always pass the fields as key-value pairs; never preformat the message with `fmt.Sprintf`.

```go
// Good
logger.Info("request completed", "method", r.Method, "path", r.URL.Path, "status", status, "duration_ms", elapsed.Milliseconds())

// Bad
logger.Info(fmt.Sprintf("request %s %s completed with status %d in %dms", r.Method, r.URL.Path, status, elapsed.Milliseconds()))
```

**Enforcement:** `golangci-lint` (`sloglint`); review.

### 8.4 — Choose the level by meaning.

**Reasoning, step by step:**
1. The level is a signal, not decoration. Pick it by what the line means for an operator, so alerts and dashboards stay meaningful:

| Level | Meaning | Example |
|-------|---------|---------|
| ERROR | Broken, needs attention now | Database connection lost, request failed after retries |
| WARN | Degraded, not broken | Retry succeeded, cache miss, fallback used |
| INFO | Business events | Request completed, job started, config loaded |
| DEBUG | Troubleshooting detail | Raw HTTP bodies, intermediate state, retry timing |

2. Put expensive serializations at `slog.LevelDebug` — they are skipped when the configured level is above DEBUG (and guard the call where the *arguments* are expensive, 8.8).

**Enforcement:** review.

### 8.5 — Mask secrets and PII with `slog.LogValuer`.

**Reasoning, step by step:**
1. Never log passwords, tokens, API keys, secret cookies, or PII (email addresses, IPs, full names, payment details, raw JWT/OAuth tokens).
2. Implement `slog.LogValuer` on sensitive types so they render a redacted representation automatically — the value is never emitted regardless of how the struct is logged:

```go
type Token struct {
    Value     string
    ExpiresAt time.Time
}

func (t Token) LogValue() slog.Value {
    return slog.StringValue("Token(***)")
}
```

**Enforcement:** review; secret-scanning in CI.

### 8.6 — Attach request context once with `slog.With`.

**Reasoning, step by step:**
1. Context that flows through a request lifecycle — request id, user id — belongs on the logger, not retyped at every call site.
2. Enrich once with `slog.With()` and pass the enriched logger downstream; don't reconstruct context at each call.

```go
logger = logger.With("request_id", requestID, "user_id", userID)
```

**Enforcement:** review.

### 8.7 — Pass the error as a structured attribute.

**Reasoning, step by step:**
1. The error belongs in a field, so it is indexable alongside the rest of the record. Always include it as a structured attribute:

```go
logger.Error("failed to authenticate", "error", err, "endpoint", endpoint)
```

2. Never use `err.Error()` as the log message — collapsing the error into the message string loses its structure.

**Enforcement:** review; `golangci-lint` (`sloglint`).

### 8.8 — Guard expensive log arguments behind a level check.

**Reasoning, step by step:**
1. When a log call's arguments require expensive computation (serialization, RPC lookups), the work runs even if the level is disabled — the arguments are evaluated before the logger decides to drop the record.
2. Guard such calls with `logger.Enabled` so the work is skipped when the level is off:

```go
// Good -- expensive serialization only runs when DEBUG is enabled
if logger.Enabled(ctx, slog.LevelDebug) {
    logger.DebugContext(ctx, "state snapshot", "state", expensiveDump())
}

// Bad -- expensiveDump() runs every time, even at INFO level
logger.Debug("state snapshot", "state", expensiveDump())
```

**Enforcement:** review.

### 8.9 — Don't log and return; log once at the boundary.

**Reasoning, step by step:**
1. If you return an error, don't also log it — the caller decides whether to log, return, or recover. Logging at both sites produces duplicate noise, one failure becoming many lines.

```go
// Bad
if err != nil {
    logger.Error("fetch config", "error", err)
    return fmt.Errorf("fetch config: %w", err)
}

// Good
if err != nil {
    return fmt.Errorf("fetch config: %w", err)
}
```

2. The exception is the **boundary** where you stop returning errors — the top of a handler, an entry point — where logging is the final action for a failure.

**Enforcement:** review.

### 8.10 — Define no command-line flags in library code.

**Reasoning, step by step:**
1. Command-line flags must only be defined in `package main` or a near-equivalent entry point. Importing a library must never register new flags as a side effect.
2. General-purpose packages are configured via Go APIs — functional options, config structs — not flags.

**Enforcement:** review; `golangci-lint` (`forbidigo`) to flag `flag.*` registration outside entry points.

### 8.11 — Never log secrets or PII; redact at the type.

**Reasoning, step by step:**
1. Never log:
   - Passwords, tokens, API keys, secret cookies.
   - Personally identifiable information (PII) — email addresses, IPs, full names, payment details.
   - Raw JWT / OAuth tokens.
2. Implement `slog.LogValuer` on sensitive types (8.5) to render a redacted representation automatically, so the secret cannot leak however the value reaches a log call.

**Enforcement:** review; secret-scanning in CI.

### 8.12 — Logging must have no side effects on control flow.

**Reasoning, step by step:**
1. Logging must never panic or affect control flow. A logging failure must not take down the process — a dropped log line is acceptable, a crashed request is not.

**Enforcement:** review.

## Cross-references

- Error wrapping with `%w` and the error chain: [chapter 03](./03-error-handling.md).
- Functional options and config structs for configuration: [chapter 05](./05-api-design.md).
- Secret masking in messages and logs: [security guide](../security.md).
</content>
</invoke>
