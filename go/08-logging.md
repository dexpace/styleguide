# 08 - Logging

## Framework

Use `log/slog` (Go 1.21+). Structured, leveled, standard library. No third-party loggers unless you need something slog doesn't provide (e.g., log rotation).

## Logger Creation

Pass `*slog.Logger` as a parameter. Never use the global default logger in library code. Applications configure the logger at startup:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
```

For library code, accept the logger as a dependency:

```go
type Client struct {
    logger *slog.Logger
    // ...
}

func NewClient(logger *slog.Logger, opts ...Option) *Client {
    return &Client{logger: logger}
}
```

## Structured Logging

Always use key-value attributes. Never `fmt.Sprintf`:

```go
// Good
logger.Info("request completed", "method", r.Method, "path", r.URL.Path, "status", status, "duration_ms", elapsed.Milliseconds())

// Bad
logger.Info(fmt.Sprintf("request %s %s completed with status %d in %dms", r.Method, r.URL.Path, status, elapsed.Milliseconds()))
```

## Log Levels

| Level | Meaning | Example |
|-------|---------|---------|
| ERROR | Broken, needs attention now | Database connection lost, request failed after retries |
| WARN | Degraded, not broken | Retry succeeded, cache miss, fallback used |
| INFO | Business events | Request completed, job started, config loaded |
| DEBUG | Troubleshooting detail | Raw HTTP bodies, intermediate state, retry timing |

Use `slog.LevelDebug` for expensive serializations -- they are skipped when the level is above DEBUG.

## Sensitive Data Masking

Never log passwords, tokens, API keys, or PII. Implement `slog.LogValuer` for sensitive types:

```go
type Token struct {
    Value     string
    ExpiresAt time.Time
}

func (t Token) LogValue() slog.Value {
    return slog.StringValue("Token(***)")
}
```

This ensures the token value is never emitted regardless of how the struct is logged.

## Context Propagation

Use `slog.With()` to add context that flows through the request lifecycle:

```go
logger = logger.With("request_id", requestID, "user_id", userID)
```

Pass the enriched logger downstream. Don't reconstruct context at every call site.

## Error Logging

Always include the error as a structured attribute:

```go
logger.Error("failed to authenticate", "error", err, "endpoint", endpoint)
```

Never use `err.Error()` as the log message -- it loses structure.

## Avoid Expensive Logs at Inactive Levels

When a log call arguments require expensive computation (serialization, RPC lookups), guard the call with a level check so the work is skipped when the level is disabled:

```go
// Good -- expensive serialization only runs when DEBUG is enabled
if logger.Enabled(ctx, slog.LevelDebug) {
    logger.DebugContext(ctx, "state snapshot", "state", expensiveDump())
}

// Bad -- expensiveDump() runs every time, even at INFO level
logger.Debug("state snapshot", "state", expensiveDump())
```

## Don't Log And Return

If you return an error, you generally should not also log it — the caller will decide whether to log, return, or recover. Logging at both sites produces duplicate noise.

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

The exception is the **boundary** where you stop returning errors — the top of a handler, an entry point — where logging is the final action for a failure.

## No Flags in Library Code

Command-line flags must only be defined in `package main` or near-equivalent entry points. Importing a library must never register new flags as a side effect. General-purpose packages are configured via Go APIs (functional options, config structs), not flags.

## PII and Secrets

Never log:
- Passwords, tokens, API keys, secret cookies.
- Personally identifiable information (PII) — email addresses, IPs, full names, payment details.
- Raw JWT / OAuth tokens.

Implement `slog.LogValuer` on sensitive types to render a redacted representation automatically.

## No Side Effects

Logging must never panic or affect control flow. A logging failure must not take down the process.
