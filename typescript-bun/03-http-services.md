# 03 — HTTP Services

The HTTP edge is where untyped, untrusted bytes become domain calls. A framework here is the contract that decides what reaches your handlers, how failures become status codes, and whether one slow client can stall the loop. This chapter fixes that contract: one framework, schema-first, thin handlers over plain domain functions, one error map, correlation on every line, limits on every edge resource. The boundary-wrap discipline ([core 8.5](../typescript/08-error-handling.md)) and zod-at-the-edge rule ([core 10.7](../typescript/10-api-design.md), BUN-3) are the raw material; here they harden into an HTTP surface.

## What good looks like

```ts
// users-route.ts — one route, done right: typed I/O, thin handler, centralized failure, correlated logs.
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';        // parse-don't-validate at the router
import { randomUUID } from 'node:crypto';
import { z } from 'zod';
import { getUser } from '../domain/users.ts';            // plain function — no Hono types in here
import { UserSchema } from '../domain/schemas.ts';       // 3.3 — the one response-shape declaration
import { runWithRequestContext } from '../observability/correlation.ts'; // AsyncLocalStorage (ch. 06)
import { mapError } from './error-map.ts';               // the one domain-error → problem+json translator

const app = new Hono();

app.use('*', async (c, next) => {                         // 3.6 — correlate before any handler runs
  const correlationId = c.req.header('x-request-id') ?? randomUUID(); // accept inbound or generate
  await runWithRequestContext({ correlationId }, next);   // store.run wraps the rest of the request (ch. 06)
});
app.onError(mapError); // 3.5 — handlers never craft 5xx; one map owns error → problem+json

app.get(
  '/users/:id',
  zValidator('param', z.object({ id: z.uuid() })),        // 3.3 — unvalidated input never reaches the body
  async (c) => {
    const { id } = c.req.valid('param');                  // 3.4 — parse → call domain → return
    return c.json(UserSchema.parse(await getUser(id)));   // 3.3 — response schema-checked on the way out
  },
);

export default { fetch: app.fetch, idleTimeout: 30 }; // 3.7 — Bun.serve picks this up; idleTimeout is the slowloris guard
```

Every byte is parsed before the handler sees it and every reply field is schema-checked on the way out (3.3); the handler is glue over `getUser`, a function the unit tests call with no HTTP in sight (3.4); failures route through one `onError` (3.5); a correlation id is generated or accepted once and threaded through `AsyncLocalStorage` so every downstream log line carries it (3.6); the server idle timeout is set explicitly on the exported `Bun.serve` config (3.7). This is the Hono-on-`Bun.serve` idiom (3.1) the rest of the chapter justifies rule by rule.

## Rules

### 3.1 — Hono on `Bun.serve` is the HTTP framework.

**Reasoning, step by step:**
1. Hono is already family-blessed — it was the edge option in the retired Node guide (tag `node-guide-final`) — and on Bun it graduates to the default. It runs directly on `Bun.serve` through a Web-standard `fetch` handler, so the same app object that serves production is the one tests invoke (3.4a). No adapter, no second server abstraction.
2. The framework choice is BUN-4's dependency-as-attack-surface reasoning applied to frameworks: prefer the one with the deepest adoption and the broadest maintenance. As of mid-2026 Hono carries ~38M weekly downloads against Elysia's ~461K — roughly 80× the adoption — with 322 contributors, a multi-runtime target (Bun, Node, Workers, Deno), and a fast release cadence. Framework throughput is not the practical bottleneck; survivability and a maintained security surface are, and that is where Hono wins.
3. Schema-first validation is parse-don't-validate built into the router: `zValidator` ([3.3](#33--every-route-declares-request-and-response-schemas)) runs *before* the handler, so unvalidated input is structurally unable to reach domain code (BUN-3). The middleware infers the validated type, which `c.req.valid('json')` reads back typed, so validator and static type are one declaration (core 10.7). Wiring is explicit values, not a DI container or reflection — exactly root rule 2 (explicit over implicit, no framework magic).

```ts
const app = new Hono();                          // Web-standard fetch handler — no container
export default { fetch: app.fetch, idleTimeout: 30 }; // served by Bun.serve directly
```

**Enforcement:** review; new HTTP services scaffold on Hono served by `Bun.serve`; route I/O is inferred from the validator via `c.req.valid`, not hand-typed.

### 3.2 — Elysia is acceptable for Bun-only experiments; Express is a legacy Node estate; NestJS is rejected.

**Reasoning, step by step:**
1. Elysia is acceptable specifically for Bun-only experiments. It is Bun-native and its Eden client gives end-to-end type-safety from server to caller, which is genuinely attractive. But its ~461K weekly downloads against Hono's ~38M, its narrower contributor base, and its single-runtime lock-in make it a deliberate bet, not the default. Reach for it when the project is Bun-only and the Eden type-safety earns its keep; for anything that may move runtimes, Hono (3.1) wins on adoption and maturity.
2. Express and Fastify are legacy Node estates only. They predate or sit outside this runtime; new Bun services do not start on them. Where a service already lives on one, it stays in the Node guide at tag `node-guide-final` — maintain it there, write nothing new on it here.
3. NestJS is rejected. Its decorator-and-DI-container model is precisely the framework magic root rule 2 forbids: dependencies resolved by reflection, invisible at the constructor, wired by metadata rather than by passing values. It is also incompatible with this family's `erasableSyntaxOnly` stance ([core type-system chapter](../typescript/03-the-type-system.md)) — Nest's constructor `parameter properties` are runtime-meaningful syntax that erasable-only TypeScript refuses to emit, and Bun strips types without type-checking, so it cannot rescue what core forbids. The two cannot coexist.

**Deviations ledger.** **Hono on `Bun.serve`** is the default (3.1). **Elysia** is acceptable for Bun-only experiments — Eden type-safety noted, adoption/maturity a deliberate bet. **Express** (and its Node-era peers) are legacy Node estates — maintain at tag `node-guide-final`, write nothing new on them. **NestJS** is rejected — its DI-container/decorator magic violates root rule 2 and its `parameter properties` break `erasableSyntaxOnly`.

**Enforcement:** review; no new routes on legacy Node frameworks under Bun, no new NestJS anywhere; Elysia confined to Bun-only experimental packages.

### 3.3 — Every route declares request AND response schemas.

**Reasoning, step by step:**
1. Each route carries a zod schema for every part of the request it reads — `param`, `query`, `json`, `header` — via `zValidator(target, schema)`. The validator runs first; a request that fails it is rejected with a 400 before the handler is entered, so unvalidated input is unreachable from domain code (BUN-3, core 10.7). The handler reads it back through `c.req.valid('json')`, typed from the same schema.
2. The response gets a schema too — a leak tripwire: a field the handler should never expose (a password hash, an internal id) fails the outbound `.parse()` in dev/test instead of shipping to a client (security.md: never leak internals). `bun test` and the dev build assert the response shape per the family contract ([05 §5.7](./05-serialization-and-validation.md)); the same `UserSchema` that types the success body validates it, so correctness comes from one declaration.

```ts
app.post(
  '/users',
  zValidator('json', CreateUserSchema),                 // request parsed before the handler runs
  async (c) => {
    const created = await createUser(c.req.valid('json'));
    return c.json(UserSchema.parse(created), 201);       // reply shape pinned — leaks fail here, not in prod
  },
);
```

**Enforcement:** review; every route declares a request validator and parses its response through a schema; a route whose response shape is not schema-checked does not pass review.

### 3.4 — Handlers are thin: parse, call a domain function, map the result.

**Reasoning, step by step:**
1. A handler does exactly three things: take the already-validated input (3.3) via `c.req.valid`, call one domain function, return its result through `c.json`. No business rules, no persistence, no branching on domain state in the handler body. The HTTP layer is glue, not logic.
2. The logic lives in plain functions — `getUser(id)`, `createUser(input)` — that take and return domain types and know nothing about `c`, `Context`, or status codes. This is data + functions (root rule 1).
3. The payoff is testability and reuse: a domain function is unit-tested value-in, value-out, at full speed, with no HTTP fixture, and is callable from a queue consumer or a CLI. A handler that embeds logic forces every test of that logic through the framework.

**3.4a — Test the app by injection through `app.request()`, not over the network.** A Hono app is a `fetch` handler, so a test invokes it directly: `app.request('/users', { method: 'POST', body, headers })` returns the `Response` with no socket opened, no port bound, no server started. This is the family's edge substitute for MSW — which `bun test` cannot run — so server-side HTTP is tested by direct invocation, in-process, deterministic. The thin-handler split (3.4) means most logic is already tested without HTTP at all; `app.request()` covers the routing, validation, and error-mapping wiring that only exists at the edge.

```ts
app.post('/orders', async (c) =>                                    // bad — logic in handler
  c.json(await db.orders.insert({ ...c.req.valid('json'), total: sum(...) })));
app.post('/orders', zValidator('json', PlaceOrderSchema), async (c) => // good — glue over a plain fn
  c.json(await placeOrder(c.req.valid('json'))));

const res = await app.request('/orders', { method: 'POST', body: JSON.stringify(order) }); // injection test — no network
```

**Enforcement:** review; handler bodies are parse-call-return; business logic in domain modules with no Hono imports; edge wiring tested via `app.request()`, never a live socket or MSW.

### 3.5 — One centralized error handler maps domain errors to responses.

**Reasoning, step by step:**
1. Exactly one `app.onError(mapError)` owns the error-to-HTTP translation. It receives every error a handler throws as `(err, c)`, walks the typed hierarchy ([core 8.1](../typescript/08-error-handling.md)) with `instanceof`, and maps each domain error to a `problem+json` response with the right status. Handlers never hand-craft a 4xx or 5xx; they throw and let the map decide.
2. This is boundary-wrap made concrete ([core 8.5](../typescript/08-error-handling.md)): the HTTP edge is the outermost boundary, so it logs once — the full `cause` chain plus the correlation id (3.6) — and no inner layer logs the same failure. The error arrives with its chain intact; the handler turns it into a response. A Hono `HTTPException` is rethrown via `err.getResponse()` so framework-level failures keep their status untouched.
3. An unexpected error — anything outside the known hierarchy — maps to a generic 500 carrying only the correlation id, with full detail in the server log. No stack trace, SQL, file path, or dependency name reaches the client (security.md). Known operational errors say what the caller did wrong; unknown ones say nothing specific.

```ts
import { HTTPException } from 'hono/http-exception';

app.onError((err, c) => {
  const correlationId = getCorrelationId();                                  // from ALS (3.6)
  logger.error({ err, correlationId }, 'request failed');                    // logged once, here
  if (err instanceof HTTPException)   return err.getResponse();              // framework status kept intact
  if (err instanceof NotFoundError)   return c.json({ type: 'not-found', correlationId }, 404);
  if (err instanceof ValidationError) return c.json({ type: 'invalid', correlationId }, 400);
  return c.json({ type: 'internal', correlationId }, 500);                   // unexpected — nothing leaks
});
```

**Enforcement:** review; one `app.onError` per app; no handler crafts a 500; client error bodies carry no internals.

### 3.6 — Every request carries a correlation id.

**Reasoning, step by step:**
1. A middleware registered with `app.use('*', ...)` either accepts an inbound `x-request-id` (so a trace spans services) or generates a fresh UUID when none arrives, via `randomUUID` from `node:crypto`. The id is decided once, at the very front of the request, before any handler or domain call.
2. It propagates through `AsyncLocalStorage` ([06](./06-logging.md)), which works unchanged on Bun, not by threading a parameter through every function. The middleware binds it with `store.run` wrapping the request continuation — `await store.run(ctx, next)` — the scoped form [06 §6.2](./06-logging.md) mandates over the mid-handler accessor it bans for leaking context into whatever runs next on the loop. Any code in the request's async context — domain function, repository, error handler — then reads the id from the store, so correlation survives `await` boundaries without polluting signatures.
3. Every log line during the request carries that id, which makes the one boundary log (3.5) joinable to all that led to it — parity with the kotlin-jvm correlation contract ([kotlin-jvm 06](../kotlin-jvm/06-logging.md)): one id, set at the edge, on every line, across the whole call tree.
4. The canonical log key is `correlationId`, set once here at the edge and read everywhere downstream ([06 §6.2](./06-logging.md)) — one name for the id in the child logger, the `AsyncLocalStorage` store, and every line, so a trace joins without reconciling synonyms.

```ts
app.use('*', async (c, next) => {
  const correlationId = c.req.header('x-request-id') ?? randomUUID(); // accept inbound or generate
  await runWithRequestContext({ correlationId }, next); // store.run wraps the request continuation, the scoped form ch. 06 §6.2 mandates
});
```

**Enforcement:** review; a `correlationId` middleware on every app; logger configured to emit `correlationId` on every line (ch. 06).

### 3.7 — Set server timeouts explicitly.

**Reasoning, step by step:**
1. A request the server waits on forever is a denial-of-service primitive. `Bun.serve`'s `idleTimeout` — how long a connection may sit without progress before Bun closes it — is set explicitly, never left default. It is the slowloris guard at the socket: a client dribbling one byte at a time is dropped when the idle window lapses.
2. Per-route work gets a ceiling too, from Hono's `timeout` middleware (`import { timeout } from 'hono/timeout'`), which rejects a handler that runs past its budget. Leaving these unset is a config bug; limits on everything (root rule 9) makes setting them mandatory. `Bun.serve`'s `idleTimeout` defaults to 10 seconds and caps at 255 — state the chosen value rather than inheriting the default silently.
3. State the defaults and why: a sane baseline is `idleTimeout: 30` seconds on the server and a per-route `timeout(30_000)` on long-running handlers — both kept under any upstream load balancer's idle timeout so the server, not the proxy, closes first. A streaming endpoint omits the route timeout (it cannot wrap a stream) and relies on the connection idle window plus an explicit `setTimeout`.

```ts
import { timeout } from 'hono/timeout';
app.use('/api/*', timeout(30_000));                  // per-route ceiling; rejects past-budget handlers
export default { fetch: app.fetch, idleTimeout: 30 };// idleTimeout default is 10s, max 255 — set it; keep under LB idle
```

**Enforcement:** review; `idleTimeout` set explicitly on the `Bun.serve` config with a comment stating the value; long-running routes carry a `timeout(...)` middleware.

### 3.8 — Rate-limit at the edge with bounded state.

**Reasoning, step by step:**
1. Public endpoints are rate-limited, either at a gateway or in-process with a Hono rate-limit middleware. An unthrottled public endpoint is an open invitation to abuse; the limiter returns `429 Too Many Requests` with a `Retry-After` header (security.md).
2. The algorithm is token-bucket (or sliding-window) — smooth, burst-tolerant, standard. What matters as much is the store: it must be bounded. A naive per-key counter in an unbounded map is itself a memory-exhaustion vector — an attacker rotates keys to grow it without limit — so the state gets a fixed cap (LRU with a max size, or shared Redis with TTL eviction; Bun's native Redis client fits the shared case). This ports security.md's rate-limiting to the runtime and obeys limits on everything (root rule 9): the defense must not become the vulnerability.

```ts
app.use('/api/*', rateLimiter({
  limit: 100, windowMs: 60_000,        // token-bucket budget per key
  store: new LruStore({ max: 10_000 }),// bounded store — keys evict; the map cannot grow without limit
  keyGenerator: (c) => c.req.header('x-forwarded-for') ?? 'anon',
}));
```

**Enforcement:** review; public endpoints rate-limited at gateway or via Hono middleware; the limiter store has an explicit size or TTL bound.

### 3.9 — Health endpoints are honest.

**Reasoning, step by step:**
1. Liveness and readiness are different questions on different endpoints. Liveness (`/health/live`) answers "is the process up?" — cheap and dependency-free, because a failing liveness check tells the supervisor to *restart*; never check a database here, since a transient blip would trigger a restart storm.
2. Readiness (`/health/ready`) answers "can this instance serve traffic now?" — it checks dependencies (database pool, message broker, downstream services) are reachable. A failing readiness check tells the load balancer to *stop routing* here, not to kill the process.
3. Readiness is also the shutdown lever: on `SIGTERM` it flips to failing *first* so the load balancer drains this instance before in-flight requests are cut, tying to the graceful-drain lifecycle ([02](./02-concurrency-and-event-loop.md)'s drain, BUN-5). An honest readiness probe is how zero-downtime deploys happen.

```ts
app.get('/health/live', (c) => c.json({ status: 'ok' }));  // process up — no dependency checks
app.get('/health/ready', async (c) =>                      // dependencies reachable — gates traffic
  shuttingDown || !(await db.isReachable())
    ? c.json({ status: 'draining' }, 503)
    : c.json({ status: 'ready' }));
```

**Enforcement:** review; separate liveness and readiness endpoints; liveness does no dependency I/O; readiness fails during shutdown drain.

## Cross-references

- Typed error hierarchies, `cause` chains, boundary-wrap (log once at the top): [core 8.1, 8.5](../typescript/08-error-handling.md).
- zod at every boundary, `z.infer` as the single source of type truth: [core 10.7](../typescript/10-api-design.md); every-edge parsing and response-shape checks: BUN-3, [05 §5.7](./05-serialization-and-validation.md).
- `erasableSyntaxOnly` and why `parameter properties` are banned: [core type-system chapter](../typescript/03-the-type-system.md).
- Correlation via `AsyncLocalStorage`, structured logs, PII redaction: [06](./06-logging.md); kotlin-jvm parity: [kotlin-jvm 06](../kotlin-jvm/06-logging.md).
- No blocking the loop in request paths, graceful drain on `SIGTERM`: [02](./02-concurrency-and-event-loop.md), BUN-5. Schema-derived serialization economics: [07](./07-bun-performance.md).
- The `app.request()` injection rule that mandates direct in-process invocation and reserves MSW for React component tests (3.4a): [core 11.4](../typescript/11-testing.md).
- Rate limiting, request-size limits, timeouts, not leaking internals: [security guide](../security.md). Limits on everything: root rule 9.
</content>
</invoke>
