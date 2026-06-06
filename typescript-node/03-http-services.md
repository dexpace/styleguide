# 03 — HTTP Services

The HTTP edge is where untyped, untrusted bytes become domain calls. A framework here is the contract that decides what reaches your handlers, how failures become status codes, and whether one slow client can stall the loop. This chapter fixes that contract: one framework, schema-first, thin handlers over plain domain functions, one error map, correlation on every line, limits on every edge resource. The boundary-wrap discipline ([core 8.5](../typescript/08-error-handling.md)) and zod-at-the-edge rule ([core 10.7](../typescript/10-api-design.md), NODE-3) are the raw material; here they harden into an HTTP surface.

## What good looks like

```ts
// users-route.ts — one route, done right: typed I/O, thin handler, centralized failure, correlated logs.
import Fastify from 'fastify';
import { randomUUID } from 'node:crypto';
import { z } from 'zod';
import { serializerCompiler, validatorCompiler, type ZodTypeProvider } from 'fastify-type-provider-zod';
import { getUser } from '../domain/users.js';            // plain function — no Fastify types in here
import { runWithRequestContext } from '../observability/correlation.js'; // AsyncLocalStorage (ch. 06)
import { mapError } from './error-map.js';               // the one domain-error → problem+json translator

const app = Fastify({
  requestTimeout: 30_000, headersTimeout: 35_000, keepAliveTimeout: 5_000, bodyLimit: 1_048_576, // 3.7 — slowloris is config
}).withTypeProvider<ZodTypeProvider>();
app.setValidatorCompiler(validatorCompiler).setSerializerCompiler(serializerCompiler);

app.addHook('onRequest', (req, _reply, done) => {         // 3.6 — correlate before any handler runs
  const id = String(req.headers['x-request-id'] ?? randomUUID());
  req.log = req.log.child({ correlationId: id });
  runWithRequestContext({ correlationId: id }, done);      // store.run wraps the rest of the request (ch. 06)
});
app.setErrorHandler(mapError); // 3.5 — handlers never craft 5xx; one map owns error → problem+json

app.get('/users/:id', {
  schema: {
    params: z.object({ id: z.uuid() }),          // 3.3 — unvalidated input never reaches the body
    response: { 200: z.object({ id: z.uuid(), email: z.email() }) }, // leaks caught here
  },
}, async (req) => getUser(req.params.id)); // 3.4 — parse → call domain → return; mapping/logging are hooks
```

Every byte is parsed before the handler sees it and every reply field is schema-checked on the way out (3.3); the handler is glue over `getUser`, a function the unit tests call with no HTTP in sight (3.4); failures route through one `setErrorHandler` (3.5); a correlation id is minted or accepted once and threaded through `AsyncLocalStorage` so every downstream log line carries it (3.6); the three server timeouts are set explicitly (3.7). This is the Fastify idiom (3.1) the rest of the chapter justifies rule by rule.

## Rules

### 3.1 — Fastify is the HTTP framework.

**Reasoning, step by step:**
1. Schema-first validation is parse-don't-validate built into the router: a route's `schema` runs *before* the handler, so unvalidated input is structurally unable to reach domain code (NODE-3). The framework enforces the boundary the language guide only asks for.
2. The plugin/decorator model is explicit wiring — `register(plugin)`, `decorate(name, value)` — with no DI container and no reflection. Dependencies are values you pass, visible at the call site, which is exactly root rule 2 (explicit over implicit, no framework magic).
3. It is fast where the edge is hottest — substantially faster than Express in like-for-like benchmarks, because validation and serialization are compiled from schemas, not interpreted per request (root rule 11) — and zod is first-class through type providers: `withTypeProvider<ZodTypeProvider>()` infers `req.params`/`req.body` from the schema, so validator and static type are one declaration (core 10.7).

```ts
const app = Fastify().withTypeProvider<ZodTypeProvider>(); // schema-first, typed, no container
```

**Enforcement:** review; new HTTP services scaffold on Fastify; the type provider is wired so route I/O is inferred, not hand-typed.

### 3.2 — Express is legacy-maintenance only; Hono is acceptable at the edge; NestJS is rejected.

**Reasoning, step by step:**
1. Express predates schema-first routing and typed providers; it stays only to maintain services already on it. New routes and new services go to Fastify (3.1), not Express.
2. Hono is acceptable specifically at the edge — Workers, Lambda, runtimes where Fastify's Node server does not fit. It shares the schema-first, no-container philosophy, so the rules below port to it; on a Node server, Fastify is still the default.
3. NestJS is rejected. Its decorator-and-DI-container model is precisely the framework magic root rule 2 forbids: dependencies resolved by reflection, invisible at the constructor, wired by metadata rather than by passing values. It is also incompatible with this guide's `erasableSyntaxOnly` stance ([core type-system chapter](../typescript/03-the-type-system.md)) — Nest's constructor `parameter properties` are runtime-meaningful syntax that erasable-only TypeScript refuses to emit. The two cannot coexist.

**Deviations ledger.** **Fastify** is the default (3.1). **Express** is legacy-maintenance only — pre-schema, maintain existing, write nothing new on it. **Hono** is acceptable at the edge — edge runtimes only, shares the no-container philosophy. **NestJS** is rejected — its DI-container/decorator magic violates root rule 2 and its `parameter properties` break `erasableSyntaxOnly`.

**Enforcement:** review; no new Express routes, no new NestJS anywhere; Hono confined to edge-runtime packages.

### 3.3 — Every route declares request AND response schemas.

**Reasoning, step by step:**
1. Each route carries a zod schema for every part of the request it reads — `params`, `querystring`, `body`, `headers` — via the type provider. The schema runs first; a request that fails it is rejected with a 400 before the handler is entered, so unvalidated input is unreachable from domain code (NODE-3, core 10.7).
2. The response gets a schema too, keyed by status code — a leak tripwire: a field the handler should never expose (a password hash, an internal id) fails serialization in tests and review instead of shipping to a client (security.md: never leak internals). It also pays at runtime — Fastify compiles it into a serializer that outruns generic `JSON.stringify` (forward-ref [07](./07-node-performance.md)) — so correctness and speed come from one declaration.

```ts
app.post('/users', {
  schema: {
    body: CreateUserSchema,                       // request parsed before the handler runs
    response: { 201: UserSchema },                // reply shape pinned — leaks fail here, not in prod
  },
}, async (req, reply) => { reply.code(201); return createUser(req.body); });
```

**Enforcement:** review; every route declares request and response schemas; a route without a `response` schema does not pass review.

### 3.4 — Handlers are thin: parse, call a domain function, map the result.

**Reasoning, step by step:**
1. A handler does exactly three things: take the already-parsed input (3.3), call one domain function, return its result. No business rules, no persistence, no branching on domain state in the handler body. The HTTP layer is glue, not logic.
2. The logic lives in plain functions — `getUser(id)`, `createUser(input)` — that take and return domain types and know nothing about `req`, `reply`, or status codes. This is data + functions (root rule 1).
3. The payoff is testability and reuse: a domain function is unit-tested value-in, value-out, at full speed, with no HTTP fixture, and is callable from a queue consumer or a CLI. A handler that embeds logic forces every test of that logic through the framework.

```ts
app.post('/orders', async (req) => db.orders.insert({ ...req.body, total: sum(req.body.items) })); // bad — logic in handler
app.post('/orders', { schema: { body: PlaceOrderSchema } }, async (req) => placeOrder(req.body));   // good — glue over a plain fn
```

**Enforcement:** review; handler bodies are parse-call-return; business logic in domain modules with no Fastify imports.

### 3.5 — One centralized error handler maps domain errors to responses.

**Reasoning, step by step:**
1. Exactly one `setErrorHandler` owns the error-to-HTTP translation. It receives every error a handler throws, walks the typed hierarchy ([core 8.1](../typescript/08-error-handling.md)) with `instanceof`, and maps each domain error to a `problem+json` response with the right status. Handlers never hand-craft a 4xx or 5xx; they throw and let the map decide.
2. This is boundary-wrap made concrete ([core 8.5](../typescript/08-error-handling.md)): the HTTP edge is the outermost boundary, so it logs once — the full `cause` chain plus the correlation id (3.6) — and no inner layer logs the same failure. The error arrives with its chain intact; the handler turns it into a response.
3. An unexpected error — anything outside the known hierarchy — maps to a generic 500 carrying only the correlation id, with full detail in the server log. No stack trace, SQL, file path, or dependency name reaches the client (security.md). Known operational errors say what the caller did wrong; unknown ones say nothing specific.

```ts
app.setErrorHandler((err, req, reply) => {
  const { correlationId } = req;
  req.log.error({ err, correlationId }, 'request failed');                       // logged once, here
  if (err instanceof NotFoundError)   return reply.code(404).send({ type: 'not-found', correlationId });
  if (err instanceof ValidationError) return reply.code(400).send({ type: 'invalid', correlationId });
  return reply.code(500).send({ type: 'internal', correlationId });              // unexpected — nothing leaks
});
```

**Enforcement:** review; one `setErrorHandler` per app; no `reply.code(500)` in handlers; client error bodies carry no internals.

### 3.6 — Every request carries a correlation id.

**Reasoning, step by step:**
1. An `onRequest` hook either accepts an inbound `x-request-id` (so a trace spans services) or mints a fresh UUID when none arrives. The id is decided once, at the very front of the request, before any handler or domain call.
2. It propagates through `AsyncLocalStorage` ([06](./06-logging.md)), not by threading a parameter through every function. The hook binds it with `store.run` wrapping the request continuation — the `done` callback — the scoped form [06 §6.2](./06-logging.md) mandates over the mid-handler accessor it bans for leaking context into whatever runs next on the loop. Any code in the request's async context — domain function, repository, error handler — then reads the id from the store, so correlation survives `await` boundaries without polluting signatures.
3. Every log line during the request carries that id, which makes the one boundary log (3.5) joinable to all that led to it — parity with the kotlin-jvm correlation contract ([kotlin-jvm 06](../kotlin-jvm/06-logging.md)): one id, set at the edge, on every line, across the whole call tree.
4. The canonical log key is `correlationId`, set once here at the edge and read everywhere downstream ([06 §6.2](./06-logging.md)) — one name for the id in the child logger, the `AsyncLocalStorage` store, and every line, so a trace joins without reconciling synonyms.

```ts
app.addHook('onRequest', (req, _reply, done) => {
  req.correlationId = String(req.headers['x-request-id'] ?? randomUUID()); // accept inbound or mint
  runWithRequestContext({ correlationId: req.correlationId }, done); // store.run wraps the request continuation, the scoped form ch. 06 §6.2 mandates
});
```

**Enforcement:** review; an `onRequest` correlation hook on every app; logger configured to emit `correlationId` on every line (ch. 06).

### 3.7 — Set server timeouts explicitly.

**Reasoning, step by step:**
1. A request the server waits on forever is a denial-of-service primitive. Three timeouts are set explicitly, never left default: `requestTimeout` (whole-request ceiling), `headersTimeout` (how long a client may dribble headers), `keepAliveTimeout` (idle persistent-connection lifetime).
2. Slowloris — a client sending one header byte per second to pin a connection open — is defeated by configuration, not code: `headersTimeout` caps the header phase and drops the connection. Leaving these unset is a config bug; limits on everything (root rule 9) makes setting them mandatory. Fastify's `requestTimeout` even defaults to `0` (off), so it *must* be set explicitly.
3. State the defaults and why: a sane baseline is `requestTimeout: 30s`, `headersTimeout` slightly above it, `keepAliveTimeout: 5s` to free idle sockets — kept under any upstream load balancer's idle timeout so the server, not the proxy, closes first.

```ts
// requestTimeout defaults to 0 (off) — set all three; headersTimeout is the slowloris guard, keepAlive < LB idle.
const app = Fastify({ requestTimeout: 30_000, headersTimeout: 35_000, keepAliveTimeout: 5_000 });
```

**Enforcement:** review; `requestTimeout`/`headersTimeout`/`keepAliveTimeout` set explicitly with a comment stating each chosen value.

### 3.8 — Rate-limit at the edge with bounded state.

**Reasoning, step by step:**
1. Public endpoints are rate-limited, either at a gateway or in-process with `@fastify/rate-limit`. An unthrottled public endpoint is an open invitation to abuse; the limiter returns `429 Too Many Requests` with a `Retry-After` header (security.md).
2. The algorithm is token-bucket (or sliding-window) — smooth, burst-tolerant, standard. What matters as much is the store: it must be bounded. A naive per-key counter in an unbounded map is itself a memory-exhaustion vector — an attacker rotates keys to grow it without limit — so the state gets a fixed cap (LRU with a max size, or shared Redis with TTL eviction). This ports security.md's rate-limiting to the runtime and obeys limits on everything (root rule 9): the defense must not become the vulnerability.

```ts
await app.register(rateLimit, {
  max: 100, timeWindow: '1 minute',  // token-bucket budget per key
  cache: 10_000,                     // bounded store — keys evict; the map cannot grow without limit
});
```

**Enforcement:** review; public endpoints rate-limited at gateway or via `@fastify/rate-limit`; the limiter store has an explicit size or TTL bound.

### 3.9 — Health endpoints are honest.

**Reasoning, step by step:**
1. Liveness and readiness are different questions on different endpoints. Liveness (`/health/live`) answers "is the process up?" — cheap and dependency-free, because a failing liveness check tells the supervisor to *restart*; never check a database here, since a transient blip would trigger a restart storm.
2. Readiness (`/health/ready`) answers "can this instance serve traffic now?" — it checks dependencies (database pool, message broker, downstream services) are reachable. A failing readiness check tells the load balancer to *stop routing* here, not to kill the process.
3. Readiness is also the shutdown lever: on `SIGTERM` it flips to failing *first* so the load balancer drains this instance before in-flight requests are cut, tying to the graceful-drain lifecycle ([02](./02-concurrency-and-event-loop.md)'s drain, NODE-5). An honest readiness probe is how zero-downtime deploys happen.

```ts
app.get('/health/live', async () => ({ status: 'ok' }));  // process up — no dependency checks
app.get('/health/ready', async (_req, reply) =>           // dependencies reachable — gates traffic
  shuttingDown || !(await db.isReachable()) ? reply.code(503).send({ status: 'draining' }) : { status: 'ready' });
```

**Enforcement:** review; separate liveness and readiness endpoints; liveness does no dependency I/O; readiness fails during shutdown drain.

## Cross-references

- Typed error hierarchies, `cause` chains, boundary-wrap (log once at the top): [core 8.1, 8.5](../typescript/08-error-handling.md).
- zod at every boundary, `z.infer` as the single source of type truth: [core 10.7](../typescript/10-api-design.md); every-edge parsing: NODE-3.
- `erasableSyntaxOnly` and why `parameter properties` are banned: [core type-system chapter](../typescript/03-the-type-system.md).
- Correlation via `AsyncLocalStorage`, structured logs, PII redaction: [06](./06-logging.md); kotlin-jvm parity: [kotlin-jvm 06](../kotlin-jvm/06-logging.md).
- No blocking the loop in request paths, graceful drain on `SIGTERM`: [02](./02-concurrency-and-event-loop.md), NODE-5. Compiled serialization: [07](./07-node-performance.md).
- Rate limiting, request-size limits, timeouts, not leaking internals: [security guide](../security.md). Limits on everything: root rule 9.
