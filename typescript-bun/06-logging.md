# 06 — Logging

Logs are the only witness to what a service did at 3am when no one was watching. On Bun, that witness has to survive a single-threaded event loop, an async call graph that loses thread-locals at every `await`, and an aggregator that ingests JSON, not prose. This chapter binds logging to pino, carries request context through `AsyncLocalStorage` (the MDC analog the JVM guide reaches for), redacts PII and secrets at the logger rather than the call site, and writes one line per request outcome at the boundary. It is the Bun port of [kotlin-jvm/06-logging.md](../kotlin-jvm/06-logging.md): MDC becomes `AsyncLocalStorage`, the masking config becomes pino `redact`.

## What good looks like

```ts
import { pino } from 'pino';
import { AsyncLocalStorage } from 'node:async_hooks';

interface RequestContext { readonly correlationId: string; readonly principal?: string; }

const store = new AsyncLocalStorage<RequestContext>();

/** Base logger. Redaction is configured once, here — never the call site's job. */
const root = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: [ // representative subset; 6.3 has the canonical list
    'req.headers.authorization', 'req.headers.cookie', 'req.headers["x-api-key"]',
    '*.password', '*.token', '*.secret', '*.cardNumber', '*.pan',
  ],
  // No `transport` key: base pino writes NDJSON straight to stdout, the Bun-safe shape (6.1).
  // Pretty-printing is a dev-terminal transport, set outside the app via `pino-pretty`; prod emits JSON.
});

/** Runs `fn` with the request context bound; every `log()` inside the async tree inherits it. */
export function runWithRequestContext<T>(ctx: RequestContext, fn: () => T): T {
  return store.run(ctx, fn);
}

/** Context-aware accessor: injects correlationId + principal without threading a logger parameter. */
export function log(): pino.Logger {
  const ctx = store.getStore();
  return ctx ? root.child(ctx) : root;
}

export async function handleCharge(req: ChargeRequest): Promise<Response> {
  return runWithRequestContext({ correlationId: req.id, principal: req.userId }, async () => {
    try {
      const receipt = await chargeCard(req.card, req.amount, req.id); // inner layers enrich, never log
      log().info({ amount: req.amount, last4: receipt.last4 }, 'charge completed');
      return Response.json(receipt);
    } catch (e: unknown) {
      log().error({ err: e }, 'charge failed'); // one line, the boundary, full cause chain
      return Response.json({ correlationId: req.id }, { status: 500 });
    }
  });
}
```

This is structured JSON through pino, running on Bun with no transport so the writes stay on the Bun-safe path (6.1); `AsyncLocalStorage` carries `correlationId` and `principal` across every `await` without a parameter (6.2); `redact` masks headers and secret fields at the logger (6.3); the inner `chargeCard` enriches via `cause` and the boundary writes exactly one outcome line (6.4, porting [../typescript/08-error-handling.md](../typescript/08-error-handling.md) §8.5); the levels are deliberate — `info` for the state transition, `error` only because it pages (6.5).

## Rules

### 6.1 — pino on Bun, structured JSON. Logs are data first.

**Reasoning, step by step:**
1. A log line is consumed by a machine — an aggregator that indexes, filters, and alerts on fields — before any human reads it. Free-form strings (`` `charged ${amount} for ${user}` ``) force that machine to parse prose back into fields, badly. Emit the fields directly: `log().info({ amount, userId }, 'charge completed')`.
2. pino is the baseline and the one logger the [overlay README](./README.md) names: structured JSON by design, low overhead because it serializes outside the hot path. It runs on Bun — Bun's `node:` compatibility covers the surface pino's core needs, and the base logger writes newline-delimited JSON straight to stdout with no worker thread. Use it; do not wrap `console` or pull a second logging library. The message string is the stable, low-cardinality event name and the variable data goes in the object — that split is what lets you alert on `msg: "charge failed"` regardless of which user it failed for.
3. The caveat is transports. pino's `transport` option (and `pino-pretty` when wired in-process) spawns a `worker_thread` to do the formatting off the main loop, and Bun's `worker_threads` support is partial — `Worker` exists but without `stdin`/`stdout`/`stderr` wiring and with documented gaps, exactly the surface a transport leans on. So on Bun do not configure an in-process transport in production. The recorded fallback holds either way, and is the default: configure **no** `transport` key. Base pino then emits the same JSON, the same fields, to stdout, and the platform's log collector (the container runtime, the sidecar, `bun run … | pino-pretty` in a dev terminal) ships and formats it out of process — where formatting belongs regardless of runtime. Gate pretty-printing on the environment outside the app; the application itself always emits JSON so prod and the aggregator agree on shape.

```ts
log().info({ orderId, itemCount }, 'order placed');             // good — event name stable, data structured
log().info(`order ${orderId} placed with ${itemCount} items`); // bad — data melted into prose, unindexable
```

**Enforcement:** review; pino as the only logging dependency; no in-process `transport` configured (base JSON to stdout, collector ships it); message strings are constant, data is the object argument.

### 6.2 — `AsyncLocalStorage` carries request context.

**Reasoning, step by step:**
1. Every log line for a request needs the same cross-cutting context: a correlation id, the principal, the tenant. Threading a `logger` or `ctx` parameter through every function to achieve that pollutes signatures all the way down and breaks the moment one layer forgets to pass it.
2. `AsyncLocalStorage` from `node:async_hooks` is the answer, and the direct analog of SLF4J's MDC in [kotlin-jvm/06-logging.md](../kotlin-jvm/06-logging.md) §6.6. Bun ships it through its `node:async_hooks` compatibility, and it works for this pattern: the store follows the async call graph across every `await`, timer, and microtask — the store set before an `await` is the store seen after it. Where the JVM bridges MDC across coroutine suspensions with `MDCContext`, here no bridging is needed.
3. Bind the store once at the boundary with `runWithRequestContext(ctx, fn)` and read it through a `log()` accessor that does `root.child(store.getStore())`. Correlation id and principal then appear on every line in that request's async subtree, no parameter passed. The canonical context key is `correlationId` — one name in the store, the child logger, and every line, so a trace joins downstream without reconciling synonyms ([03 §3.6](./03-http-services.md) mints it at the edge). Use `store.run` to scope it; never `enterWith` mid-handler, which leaks the context into whatever runs next on the loop.

```ts
app.use('*', (c, next) => runWithRequestContext({ correlationId: c.req.header('x-request-id') ?? randomUUID(), principal: c.get('principal') }, next));
log().warn({ retries }, 'gateway slow, retrying'); // 12 frames deep, no logger threaded — still carries correlationId
```

**Enforcement:** review; context bound via `runWithRequestContext` at the boundary, reads through `log()`, no logger threaded as a parameter.

### 6.3 — `redact` paths for PII and secrets at the logger.

**Reasoning, step by step:**
1. Masking that depends on every call site remembering to strip a field will fail the one time it matters — a new endpoint logs the raw request, and a password is in the aggregator forever. Masking is *configuration*, declared once where the logger is built, not discipline repeated at hundreds of call sites. This is the Bun port of the masking rules in [security.md](../security.md); see them for the full forbidden-field policy.
2. pino's `redact` option takes a list of paths and replaces matching values with `[Redacted]` before serialization. Configure the canonical set on the base logger so it applies to every child and every line, including objects you logged without realizing they held a secret.
3. The canonical paths cover the headers and fields [security.md](../security.md) names. List them explicitly — authorization, cookie, set-cookie, x-api-key, and proxy-authorization headers, plus the secret-bearing body fields:
```ts
redact: [
  'req.headers.authorization', 'req.headers.cookie', 'req.headers["set-cookie"]',
  'req.headers["x-api-key"]', 'req.headers["proxy-authorization"]',
  '*.password', '*.token', '*.secret', '*.cardNumber', '*.pan', '*.cvv',
]
```
4. Redaction is the safety net, not a licence to log secrets and trust the net. Still never deliberately pass a token or PAN to a log call — mask to a fragment at the source (`card ****1234`) per [../typescript/08-error-handling.md](../typescript/08-error-handling.md) §8.8. `redact` catches what slips through; the call site is the first line.

**Enforcement:** review; `redact` configured on the base logger with the canonical paths; secret-scanning in CI per [security.md](../security.md).

### 6.4 — Log once, at the boundary.

**Reasoning, step by step:**
1. If every layer an error passes through logs it, one failure becomes five lines in the aggregator and the real signal drowns in its own echo. Inner layers do not log; they enrich the error with context and let it propagate, wrapping with `cause` per [../typescript/08-error-handling.md](../typescript/08-error-handling.md) §8.5. The outermost boundary — the HTTP handler, the queue consumer, the cron entry — writes exactly one line per request outcome: success at `info`, failure at `error` or `warn`, with the full `cause` chain and the correlation id already on the line via the request context (6.2).
2. Logging in a `catch` block is therefore a boundary-only act. A `catch` in a domain or repository layer that logs and rethrows is duplicate logging; it either handles (and may log) or wraps-and-rethrows (and stays silent). Pass the whole error object — `log().error({ err: e }, 'msg')` — and let pino's serializer render the chain; never log `e.message` alone and lose the stack and cause.

```ts
catch (e: unknown) { throw new OrderPersistFailedError(order.id, { cause: toError(e) }); } // inner: enrich, silent
catch (e: unknown) { log().error({ err: e }, 'order request failed'); return fail(req.id); }  // boundary: the one log
```

**Enforcement:** review; logging calls in `catch` blocks live only in boundary modules; whole-error objects, not `.message`.

### 6.5 — Levels mean something.

**Reasoning, step by step:**
1. The level is a routing instruction for a human under pressure, not a vague vibe. **error** means page someone — act now. **warn** means degraded but still serving — a fallback fired, a retry succeeded, capacity is tight; review it, don't wake to it. **info** marks state transitions worth a forensic trace — request received, order placed, job completed — bounded in volume. **debug** is diagnosis detail, off in production.
2. Level inflation is corrosive. Log a user's mistyped 404 or a routine validation rejection at `error` and the on-call learns that `error` is noise, then sleeps through the page that was real. Calibrate down: an expected operational outcome (a declined card, a not-found) is `info` or `warn`, never `error` — `error` is reserved for a failure the program could not handle. `debug` is gated by `level` and stays off in prod (`LOG_LEVEL=info`); if `info` exceeds a handful of lines per request, events that belong at `debug` have crept up a level.

```ts
log().info({ orderId }, 'order placed');                // state transition
log().warn({ attempt, max }, 'gateway retry');          // degraded, still serving
log().error({ err }, 'gateway unreachable, giving up'); // page-worthy: handling failed
```

**Enforcement:** review; `error` reserved for page-worthy failures, expected outcomes at `info`/`warn`, `debug` off in prod via `LOG_LEVEL`.

### 6.6 — `console.*` is banned in server code.

**Reasoning, step by step:**
1. `console.log` and its siblings are everything a server log must not be: unstructured (a string, not indexable fields), unleveled (no `error`/`warn`/`info` to route on), uncorrelated (no request context), and written sync-ish to stdout — a blocking write on the single thread (6.1, [overlay §BUN-2](./README.md)). It bypasses pino's redaction (6.3) entirely, so `console.log(req.body)` leaks whatever the body holds.
2. Diagnostics go through the `log()` accessor so they inherit structure, level, correlation, and redaction. There is no server-code case where `console` is right; the one narrow exception is a CLI's *intended program output* to stdout — data the user asked for, not a diagnostic — and even there diagnostics use the logger. The ban is mechanical, not a matter of remembering: this Bun overlay carries the `'no-console': 'error'` rule on top of the core ESLint overlay (which carries only the three caps, [../typescript/01-formatting-and-tooling.md](../typescript/01-formatting-and-tooling.md) §1.7-§1.8) for server packages — the same way the React overlay layers its plugins on the core config — so a `console.*` call fails lint and never reaches review.

```ts
console.log('charging', req.body);                 // banned — unstructured, uncorrelated, unredacted
log().info({ amount: req.amount }, 'charging');    // correct
```

**Enforcement:** `no-console` added by this Bun overlay on top of the core ESLint overlay for server packages; CI fails on any `console.*` in server code.

### 6.7 — Child loggers per component.

**Reasoning, step by step:**
1. When an incident narrows to "the billing path is throwing", the first move is to filter the logs to billing. That filter only exists if every billing line carries a `component` field. A child logger binds it once — `logger.child({ component: 'billing' })` — and every line through it inherits it, so provenance is grep-able: `component: "billing"` turns the firehose into the one subsystem you care about, without parsing module names out of free text.
2. The child composes with the request context from 6.2 — a line carries both `correlationId` and `component`. Create it at module or class construction, not per call; it is cheap to make once and wasteful to recreate on every log. Bind only stable, low-cardinality provenance fields (`component`, `subsystem`) on the child, and keep per-event, high-cardinality data (ids, counts) in the call's object argument.

```ts
const log = root.child({ component: 'billing' }); // once, at module scope
log.info({ invoiceId }, 'invoice issued');         // line carries component: 'billing'
```

**Enforcement:** review; subsystems log through a `child({ component })`; provenance fields on the child, per-event data on the call.

### 6.8 — Log lines are bounded.

**Reasoning, step by step:**
1. A log is not a payload store. Dumping a full request body, a query result set, or a serialized object into a line bloats the aggregator, buries the fields that matter, and risks smuggling unredacted PII into the record. Log the identifying handles — ids, counts, a status — not the contents, and truncate every field that could be large at a stated cap: a string clipped to N characters, an array logged as its length, an object reduced to the few keys that aid diagnosis. "Limits on everything" ([root rule 9](../README.md)) applies to log fields as much as to queues and buffers.
2. The bound matters most exactly when it is most tempting to remove it: during an incident, when a tight loop or a retry storm is firing. An unbounded log line inside that loop becomes a log *flood* that saturates disk, stalls the stdout write, and amplifies the outage it was meant to diagnose. The cap is what keeps logging from becoming the second failure.

```ts
log().info({ body: req.body, rows }, 'query returned');                       // bad — unbounded, may leak PII
log().info({ rowCount: rows.length, sample: rows.slice(0, 3) }, 'query done'); // good — bounded handles
```

**Enforcement:** review; no payload dumps; large fields truncated at a stated cap; arrays logged by length or a bounded sample.

## Cross-references

- Log once at the boundary, enrich via `cause` inward, mask secrets in messages: [../typescript/08-error-handling.md](../typescript/08-error-handling.md) §8.5, §8.8.
- The MDC analog this chapter ports — `AsyncLocalStorage` for `MDCContext`: [kotlin-jvm/06-logging.md](../kotlin-jvm/06-logging.md) §6.6.
- Secret and PII masking policy, forbidden fields and headers: [security.md](../security.md).
- The event-loop budget that makes synchronous `console` writes costly: [02-concurrency-and-event-loop.md](./02-concurrency-and-event-loop.md); request correlation ids at the HTTP boundary: [03-http-services.md](./03-http-services.md); limits on everything, applied to log volume: [root rule 9](../README.md).
