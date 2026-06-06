# 05 — Serialization and Validation

Serialization is where the runtime meets everything it does not control: the process environment, request and response bodies, queue messages, third-party JSON. The core guide put a zod schema at the API boundary and derived the type with `z.infer` ([core 10.7](../typescript/10-api-design.md)); this chapter extends that discipline to every edge a Bun process touches, environment included (BUN-3), and fixes the wire encodings — time, null-versus-absent, money, binary — that a long-running service cannot afford to get wrong at 3am.

## What good looks like

```ts
// config.ts — Bun.env is parsed once, at startup, into a frozen typed config.
import {z} from 'zod';

const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.url(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  SHUTDOWN_DEADLINE_MS: z.coerce.number().int().positive().default(10_000),
});

export type Config = Readonly<z.infer<typeof EnvSchema>>;

function loadConfig(env: Record<string, string | undefined>): Config {
  const result = EnvSchema.safeParse(env);
  if (!result.success) {
    // Every missing/invalid var at once — not just the first.
    const tree = JSON.stringify(z.treeifyError(result.error), null, 2);
    throw new Error(`invalid environment:\n${tree}`); // boot dies here, by design
  }
  return Object.freeze(result.data);
}

export const config: Config = loadConfig(Bun.env); // parsed once, at module load
```

The schema runs exactly once, at startup, and a single failure lists *every* problem before the process exits (5.1) — a server never starts half-configured. The type is `z.infer` of the schema, frozen and `Readonly`, so the validator and the type cannot drift (5.2, [core 10.7](../typescript/10-api-design.md)) and nothing mutates the config after boot. `Bun.env` enters as the untyped thing it is and is parsed, never read field-by-field with raw string access (5.3). Defaults live in the schema, in one place; the boundary is the only place `env` is touched.

## Rules

### 5.1 — Parse `Bun.env` once, at startup, into a frozen typed config; fail the boot with every problem listed.

**Reasoning, step by step:**
1. The environment is the first untrusted boundary a process crosses, and it is read at the worst possible time to discover it is wrong — lazily, deep in a request, hours after deploy. So parse it *once*, at module load, through a zod schema, into a single frozen `Readonly` config object. Read it through `Bun.env`, the idiomatic Bun accessor; `process.env` also works on Bun and the two stay in sync, but `Bun.env` is the one read this guide uses. Every other module imports the parsed config object; none reads `Bun.env` (or `process.env`) directly. The boundary is one place, crossed one time (BUN-3).
2. When the environment is invalid the boot *fails* — a thrown error, a non-zero exit, before the server binds a port. A service that starts half-configured does not fail now; it fails at 3am, mid-request, in a state no one designed (fail fast, BUN-3). Crashing at boot is the supervisor's cue to hold the old version.
3. The failure lists *every* missing or invalid variable at once, not just the first. `z.treeifyError(error)` (or aggregating `error.issues`) turns one boot attempt into the complete repair list; reporting only the first var means N deploys to fix N typos. Aggregate, print, exit.

```ts
const result = EnvSchema.safeParse(Bun.env);    // Bun.env is the idiomatic read; process.env mirrors it
if (!result.success) {
  console.error(z.treeifyError(result.error)); // all problems, not just issues[0]
  process.exit(1);                             // dead at boot, not degraded at runtime
}
const config = Object.freeze(result.data);
```
**Enforcement:** review; one `config` module owns the single `Bun.env` read, schema-parsed and frozen at load; a `Bun.env.`/`process.env.` access anywhere else is a finding (lint via `no-process-env` outside the config module).

### 5.2 — Every request and response body has a zod schema, and `z.infer` is its type.

**Reasoning, step by step:**
1. A request body arrives as `unknown` over the wire and must be parsed into a domain type before a handler touches it — this is [core 10.7](../typescript/10-api-design.md) and the route rule of [core 03](../typescript/03-the-type-system.md), unchanged. The Bun layer only adds that *every* body qualifies: inbound request, outbound response, and the message on every queue between them (BUN-3).
2. The TypeScript type is `z.infer` of the schema, never a hand-written `interface` maintained beside it. Two declarations drift the moment one is edited and the other forgotten, and the drift compiles silently. One inferred type cannot drift from itself.
3. Parse at the handler edge, then trust the type inside. The schema runs once where the body enters; the rest of the request path consumes the inferred type with no re-validation, wrapped `Readonly` so it is immutable from birth (3.10).

```ts
const CreateUser = z.object({email: z.email(), name: z.string().min(1)}).readonly();
type CreateUser = z.infer<typeof CreateUser>;        // type follows schema — no hand-written twin
function handle(raw: unknown): CreateUser {
  return CreateUser.parse(raw);                       // the unknown body, parsed at the edge
}
```
**Enforcement:** review; request/response bodies are `z.infer`, never hand-written beside a schema; handler input typed `unknown` then parsed ([core 10.7](../typescript/10-api-design.md), [core 03](../typescript/03-the-type-system.md)).

### 5.3 — Raw `JSON.parse` never escapes a boundary module; parse, validate, and type in one move.

**Reasoning, step by step:**
1. `JSON.parse` returns `any`, and `any` is the absence of a type — banned by [core 03's 3.2](../typescript/03-the-type-system.md). Its result is `any` wearing a JSON disguise (BUN-3): assign it to a typed variable and the `no-unsafe-assignment` rule fires, because nothing was actually checked. The bare call is structurally valid and semantically unknown.
2. So `JSON.parse` is confined to the boundary module that owns the parse, and its output is fed straight into a zod schema in the same move — `Schema.parse(JSON.parse(text))` — so a *typed* value, never the `any`, is what leaves the function. The narrowest correct annotation for the intermediate is `unknown`, not `any`, and zod erases even that.
3. The same holds for every other producer of `any` at the edge: `res.json()`, a driver's row, a deserialized cache entry. Each is `unknown` until a schema proves otherwise; none reaches the interior unparsed (5.2).

```ts
const text: string = await readBody(req);
const order: Order = OrderSchema.parse(JSON.parse(text)); // any is born and dies inside one line
return order;                                             // only the typed value escapes
```
**Enforcement:** `@typescript-eslint/no-unsafe-assignment` and the `no-unsafe-*` family flag an unparsed `JSON.parse`; review confines the call to boundary modules.

### 5.4 — Dates are ISO-8601 strings on the wire; `Temporal` natively, `date-fns` until then, never `moment`.

**Reasoning, step by step:**
1. A timestamp on the wire is an ISO-8601 string, parsed and validated at the boundary with `z.iso.datetime()`, never an epoch number whose unit (seconds? millis?) is undocumented and whose meaning a reader must guess. The string is unambiguous, sorts lexically, and carries its offset. This is the wire parity of the JVM guide's time-type rule ([kotlin-jvm 05](../kotlin-jvm/05-serialization.md)).
2. In the domain, behind that boundary, use `Temporal` where the runtime provides it natively — `Temporal.Instant` for a point in time, `Temporal.PlainDate` for a timezone-free calendar date. Until `Temporal` is available across the deployed Bun versions, use `date-fns`: explicit, immutable, tree-shakeable functions over a `Date`.
3. Never `moment`. It is a frozen project by its own maintainers' declaration, its objects are mutable, and it pulls a heavy bundle no service should carry. A new dependency on it fails BUN-4's justification on every count.

```ts
const EventSchema = z.object({occurredAt: z.iso.datetime()}); // ISO-8601 string at the boundary
// domain side, once Temporal is native:
const at = Temporal.Instant.from(event.occurredAt);           // immutable, unambiguous
```
**Enforcement:** review; `z.iso.datetime()` on wire timestamps, never a bare epoch number; `moment` blocked at dependency review (BUN-4).

### 5.5 — Null versus absent is a contract decision, stated per field.

**Reasoning, step by step:**
1. JSON carries both `null` and a missing key, and they mean different things — most sharply in a PATCH, where `null` means "set this to null" and absent means "leave it unchanged." The distinction is a contract decision made deliberately *per field*, documented at the field, not an accident of which the serializer happened to emit ([kotlin-jvm 05](../kotlin-jvm/05-serialization.md)).
2. `exactOptionalPropertyTypes` ([core 03's 3.6](../typescript/03-the-type-system.md)) makes the distinction real in the type: `{name?: string}` permits the key's absence, while `{name: string | undefined}` *requires* the key with a possibly-undefined value. The two are no longer interchangeable, which is exactly right.
3. In zod, `.optional()` and `.nullable()` are chosen, not defaulted into. `.optional()` models an absent key, `.nullable()` an explicit `null`, and `.nullish()` both; a PATCH field that must distinguish the two uses `.optional()` over a `.nullable()` value. Where `null` carries no distinct meaning, convert it to `undefined` at this boundary so the interior speaks one absence ([core 03's 3.5](../typescript/03-the-type-system.md)); where it does (a PATCH that clears a field), keep `.nullable()` and say so.

```ts
const Patch = z.object({
  displayName: z.string().optional(),          // absent = unchanged
  deletedAt: z.iso.datetime().nullable(),      // explicit null = cleared; present = set
});
```
**Enforcement:** `exactOptionalPropertyTypes` ([core 03's 3.6](../typescript/03-the-type-system.md)) makes the absent/undefined misuse a compile error; review that each field's `.optional()`/`.nullable()` choice matches the documented contract.

### 5.6 — Unknown fields: strip by default, reject where integrity matters; passthrough is never silent.

**Reasoning, step by step:**
1. zod strips unknown keys by default — a forgiving read that keeps a consumer from breaking when a producer adds a field, the right default for forward compatibility on most reads. The parsed value carries only the schema's keys; the extras are dropped, not carried along as untyped baggage.
2. Where the extra key signals a client bug or a tampering risk — a command, a write, a money-moving request — strip is too quiet. Use `z.strictObject()` so an unexpected key is a parse error, not a silent drop: on a write you want to *know* the caller sent a field you did not honour, before it costs an audit.
3. `z.looseObject()`, which carries unknown keys through untyped, is never the silent default. It reintroduces the unvalidated baggage the schema exists to remove; permit it only at a deliberate proxy boundary, named and commented, where forwarding an opaque body is the actual job.

```ts
const Read = z.object({id: z.string()});            // strips unknown keys — forgiving read
const Command = z.strictObject({amount: z.number()}); // unknown key = error on a write
```
**Enforcement:** review; `z.strictObject()` on commands and writes, default strip on reads; `z.looseObject()` only at a commented proxy boundary.

### 5.7 — Outbound serialization is schema-checked too.

**Reasoning, step by step:**
1. Validation is symmetric: you cannot trust what you did not check, in either direction. A response assembled from database rows and computed fields is just as capable of being malformed — a leaked internal field, a `null` where the contract promised a value — as an inbound body. The boundary is checked on the way out as well as in.
2. In development and test, the response is run through its schema before it is sent, so a contract violation fails a test rather than reaching a client. This is also where over-posting is caught: a response schema that does not include `passwordHash` makes leaking it a parse failure, not a production incident. The `app.request()` injection test ([03](./03-http-services.md)) drives the route end to end, so the response parse runs on the real assembled body, not a fixture.
3. The outbound schema is the same kind of zod schema Hono's validator middleware applies inbound ([03](./03-http-services.md)) — one declaration, used in both directions. On a hot path where a per-response parse is too expensive to keep in prod, gate it behind a dev/test build so the contract is asserted everywhere the tests run while costing nothing on the production request; the schema that defines the shape is the same either way.

```ts
// dev/test: assert the contract before sending — a leaked field fails here, not in prod
const body = UserResponse.parse(assembleUser(row));
return c.json(body); // Hono on Bun.serve; the same schema Hono validates inbound (03) guards the outbound shape
```
**Enforcement:** review; response bodies parsed against their schema in dev/test (driven by `app.request()` injection tests, [03](./03-http-services.md)); outbound shape is checked, not assumed.

### 5.8 — BigInt, binary, and money get explicit wire encodings.

**Reasoning, step by step:**
1. JSON's `number` is an IEEE-754 double, and three common values do not survive it. Money is the sharpest: floating point is not a currency type — `0.1 + 0.2 !== 0.3`, and rounding error in a balance is a defect, not a tolerance. Money crosses the wire as an integer count of minor units or as a decimal string, and lands in the branded integer `Cents` of [core 05](../typescript/05-functions.md), never a float.
2. A 64-bit integer (`bigint`, a database `BIGINT` id, a Snowflake) exceeds `Number.MAX_SAFE_INTEGER` and silently loses its low bits when forced through a JSON `number`. It crosses as a *string* and is parsed back to `bigint` at the boundary; `z.coerce.bigint()` does the parse. The id that round-trips wrong is the bug you find by reconciliation, weeks later.
3. Binary has no JSON representation at all, so it is base64-encoded into a string and decoded at the boundary — the encoding stated in the schema, never a raw buffer smuggled through a field typed `string`. Each of these is a deliberate, documented wire encoding, validated like every other field (5.2).
4. The other side of that boundary is the byte store, and on Bun it is native: `Bun.file(path).bytes()` yields a `Uint8Array` (`.arrayBuffer()` an `ArrayBuffer`), `Bun.write` takes one back, and `s3.file(key)` (the `S3Client` from `bun`) gives the same `Blob`-shaped `.bytes()` for object storage. The schema field stays base64; these handles are where the decoded bytes live, never smuggled through JSON themselves.

```ts
const Invoice = z.object({
  amount: z.string().regex(/^\d+$/).transform(BigInt), // minor units as string → bigint; never a float
  attachmentId: z.coerce.bigint(),                     // i64 id crosses as a string
  thumbnail: z.base64(),                               // binary as base64, decoded at the edge
});

// the binary boundary: base64 on the wire ↔ native bytes at rest, decoded once.
const bytes = Buffer.from(Invoice.parse(raw).thumbnail, 'base64'); // base64 string → bytes
await Bun.write(Bun.file(`/blobs/${id}.bin`), bytes);              // Bun.file/Bun.write own the byte store
const stored = await s3.file(`thumbs/${id}`).bytes();             // s3.file (S3Client from 'bun') → Uint8Array
```
**Enforcement:** review; money as integer minor units or decimal string into `Cents` ([core 05](../typescript/05-functions.md)), i64 as string, binary as base64 — never a JSON `number` for any of the three.

## Cross-references

- zod at the boundary, `z.infer` as the single source of type truth: [core 10.7](../typescript/10-api-design.md). Boundary route rule, `unknown` inward, `any` banned, `undefined` over `null`: [core 03's §3.2, §3.5, §3.6](../typescript/03-the-type-system.md).
- Branded `Cents`, integer minor units, the parse-mint constructor: [core 05](../typescript/05-functions.md).
- Null-versus-absent and time types, JVM parity: [kotlin-jvm serialization](../kotlin-jvm/05-serialization.md).
- Parse every boundary, crash-only boot, dependency justification: BUN-3, BUN-1, BUN-4 ([README](./README.md)). Rows parsed at the edge: [persistence](./04-persistence.md). HTTP body limits and handlers: [HTTP services](./03-http-services.md).
