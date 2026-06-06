# 04 — Persistence

The database is the slowest resource a request touches and the largest untrusted surface it reads from. This chapter binds the data layer to two core spines: every row is external input parsed at the edge before it crosses inward ([core 10.7](../typescript/10-api-design.md), NODE-3), and every database fault wraps into a domain error before a caller sees it ([core 8.5](../typescript/08-error-handling.md)). The shape is deliberate — Drizzle for a thin, SQL-visible, codegen-free data layer; repositories as plain functions over a `db` handle, not classes; transactions as explicit scopes; pools sized and bounded like the [Kotlin HikariCP rule](../kotlin-jvm/04-persistence.md); and N+1 treated as a bug the way [performance.md](../performance.md) treats it. Data and functions, not objects, all the way to the rows.

## What good looks like

```ts
import {pgTable, uuid, text, integer} from 'drizzle-orm/pg-core';
import {eq, inArray, gt, sql} from 'drizzle-orm';
import {z} from 'zod';
import type {Db, Tx} from './db.js'; // Db = NodePgDatabase<typeof schema>; Tx = its transaction handle
// The schema is the SQL, visible in code (4.1); types derive from it, no codegen step.
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  credits: integer('credits').notNull().default(0),
});
// Rows are external input, parsed at the repo boundary into a domain type whose shape follows the schema (4.8, NODE-3):
const UserRow = z.object({id: z.uuid(), email: z.email(), credits: z.number().int()});
export type User = z.infer<typeof UserRow>;            // single source of type truth
const toUser = (row: unknown): User => UserRow.parse(row);
// Repositories are plain functions over a db handle (4.2), faked by swapping the handle — no mocks.
export async function findUser(db: Db, id: string): Promise<User | undefined> {
  const [row] = await db.select().from(users).where(eq(users.id, id)).limit(1); // bounded (4.6)
  return row === undefined ? undefined : toUser(row);
}
export async function findUsersByIds(db: Db, ids: readonly string[]): Promise<User[]> {
  if (ids.length === 0) return [];                                                        // batch, not a loop (4.7)
  return (await db.select().from(users).where(inArray(users.id, [...ids]))).map(toUser);  // one query, not N
}
const debit = (tx: Tx, id: string, by: number) => tx.update(users).set({credits: sql`${users.credits} - ${by}`}).where(eq(users.id, id));
const credit = (tx: Tx, id: string, by: number) => tx.update(users).set({credits: sql`${users.credits} + ${by}`}).where(eq(users.id, id));
// Transactions are explicit scopes; tx threads down to each write (4.2), never an ambient global (4.3).
export async function transfer(db: Db, from: string, to: string, amount: number): Promise<void> {
  await db.transaction(async (tx) => {                   // both writes commit or neither does
    await debit(tx, from, amount); await credit(tx, to, amount);
  });
}
```

`users` makes the SQL legible in TypeScript and `User` is `z.infer` of the row schema, so the validator and the type cannot drift (4.1, 4.8). The repository is free functions over a `Db`/`Tx` handle, never a class with a hidden connection (4.2); `findUser` and `findUsersByIds` both bound their result and the batch reader takes the whole id set in one `inArray` query rather than looping (4.6, 4.7). `transfer` opens one explicit transaction and passes `tx` to each write, so transactionality is visible at the call site (4.3). Every row crosses the boundary through `toUser`, parsed before it enters the domain (4.8). Database faults — a unique-violation on `email`, a pool timeout — wrap into typed domain errors one layer out (4.9), so this module's callers never import a driver type.

## Rules

### 4.1 — Drizzle is the data layer; raw `pg` for hot paths; Prisma needs written justification.

**Reasoning, step by step:**
1. The default is Drizzle, and the reasoning is a ledger of the core spine. *Explicit over implicit:* the query in the code is the SQL that runs — `db.select().from(users).where(eq(users.id, id))` is one legible statement, not an object graph emitting a query you cannot see at the call site. *Types from the schema, in TypeScript:* `User` is `z.infer` / `InferSelectModel` of the table definition, derived by the compiler with no codegen engine binary to run, version, and sync. *A thin runtime:* Drizzle is a query builder, not a query engine — a small, predictable layer over the driver, which matters because the database is already the slowest resource ([performance.md](../performance.md)).
2. Raw `pg` (parameterized, always) is acceptable for a measured hot path where the builder's overhead or a hand-tuned query wins — but the call still parses its rows (4.8) and wraps its errors (4.9) like any other.
3. Prisma requires a written justification in the PR, because it brings the two things this rule spends its budget avoiding: a codegen step (a generated client to regenerate and commit) and a separate query engine (a binary running queries out of process, hiding behaviour the core guide wants visible). The justification names what Prisma buys that pays for that hidden machinery.

```ts
const active = await db.select().from(users).where(eq(users.credits, 0)).limit(100); // the query you read is the SQL that runs
```

**Enforcement:** review; Drizzle by default, raw `pg` parameterized and confined to documented hot paths, Prisma blocked without a written justification in the PR.

### 4.2 — Repositories are plain functions taking a `db` handle, not classes.

**Reasoning, step by step:**
1. Root rule 1 is data and functions, not objects, and a repository is the test case. A repository is a set of queries over a connection; it owns no lifecycle of its own (the pool does — chapter 13). So it is free functions — `findUser(db, id)`, `insertUser(db, input)` — each taking the `Db` (or `Tx`) handle as its first argument, not methods on a `UserRepository` class that hides a connection in a field.
2. Functions compose and fake trivially. `findUser(db, id)` takes a real pooled handle in production and an in-memory or transaction-rolled-back handle in a test — no mock framework, just a different first argument. An active-record `user.save()` reaching a hidden global is the opposite: untestable without stubbing the global, and silent about which connection it touches.
3. Group the functions in a module by resource (`users.repo.ts`), export what the service needs, keep row mappers and query fragments private. The module is the unit of organization; the class adds a `this` that buys nothing here.

```ts
export async function insertUser(db: Db, input: NewUser): Promise<User> { /* ... */ } // good — handle is the first arg
class UserRecord { async save(): Promise<void> { /* which connection? */ } } // bad — active record hides a global
```

**Enforcement:** review; repositories are exported functions taking a `db`/`tx` handle, no repository classes, no active-record `save()`/`delete()` on rows.

### 4.3 — Transactions are explicit scopes; `tx` threads down, never ambient.

**Reasoning, step by step:**
1. A transaction is `db.transaction(async (tx) => { ... })` and nothing else. The scope is visible at the call site: the writes that are atomic are exactly the ones inside the callback, and they receive the `tx` handle. Transactionality is a property you point at in the code, not a fact you infer from a decorator three files away.
2. Pass `tx` to every participating function. A repository function running inside a transaction takes the `Tx` handle as its first argument exactly as it would take `Db` (4.2) — `debit(tx, id, amount)` — so the same function works inside or outside a transaction by the handle it is given. Never reach for a global `db` from inside a transactional flow; the write escapes the transaction silently and lands outside it.
3. This rules out ambient, decorator-driven transactionality (`@Transactional` on an invisible thread-local). On a single-threaded event loop with interleaved async work there is no safe thread-local to hang it on, and the implicit version hides the one thing that most needs to be explicit: the boundary of atomicity. Keep transactions short — one holds a pooled connection (4.5) and a row lock for its whole duration.

```ts
await db.transaction(async (tx) => {      // the atomic boundary is exactly this callback; tx passed to each write
  const order = await insertOrder(tx, input);            // not a global db — the handle threads down (4.2)
  await decrementStock(tx, order.sku, order.qty);
});
```

**Enforcement:** review; multi-write atomic flows use `db.transaction`, participating functions take `tx`, no ambient/decorator transactions, no global handle reached from inside a scope.

### 4.4 — Migrations are versioned, forward-only, reviewed SQL.

**Reasoning, step by step:**
1. The schema is code, so its changes go through review as code. `drizzle-kit` generates a migration from the schema diff, and the generated SQL is committed and read by a human before it merges — the tool writes the first draft, a person owns the result. A migration nobody read is a production change nobody reviewed.
2. Migrations are forward-only in production. There is no `down` run against live data; a mistake is corrected by writing a new forward migration that fixes it, not by reversing the last one (a rollback that drops a column discards the rows written since). This is the same forward-only discipline the [Kotlin guide](../kotlin-jvm/04-persistence.md) holds for Flyway.
3. Never auto-sync the schema in production. `drizzle-kit push` (apply-the-diff-directly, no migration file) is a development convenience only; production applies committed, reviewed migration files in order, run as an explicit step in deploy, never as an import side effect of the app booting (NODE-5). Test the migration against a representative dataset in CI on a fresh database before it ships.

```ts
// $ drizzle-kit generate → migrations/0007_*.sql (human reads + commits the SQL); migrate applies them in order at deploy, never on boot
```

**Enforcement:** review of every generated migration's SQL; forward-only (no `down` in prod, fixes are new migrations); no `push`/auto-sync in production; CI runs migrations on a fresh database.

### 4.5 — Connection pools are explicitly sized, bounded, with an acquisition timeout.

**Reasoning, step by step:**
1. The pool is configured explicitly, never left on driver defaults — the default is tuned for "works on a laptop," not for your instance count and your database's connection ceiling ([performance.md](../performance.md), pooling). State `max` (e.g. 10 per instance), sized from the bottleneck the way the [Kotlin HikariCP rule](../kotlin-jvm/04-persistence.md) does: `min(db_max_connections / instance_count, peak_concurrent_queries)`. Ten instances each opening an unbounded pool against a Postgres capped at 100 connections is an outage waiting for traffic.
2. Acquisition is deadline-bounded. Set `connectionTimeoutMillis` (e.g. 5s): when every connection is checked out, a request waits at most that long and then fails with a typed error (4.9) rather than parking on the pool forever, turning starvation into unbounded latency for every caller. This is root rule 9 — bound everything, fail fast — applied to the pool: exhaustion is a fast, visible failure, not a silent hang.
3. The pool is a named lifecycle resource (NODE-5, chapter 13): opened once at startup, shared as the singleton every repository function receives (never per request — that defeats pooling, [performance.md](../performance.md)), closed on `SIGTERM` during drain. Set `idleTimeoutMillis` and `maxLifetimeSeconds` so idle connections are reclaimed and long-lived ones rotate off stale server state. Document the numbers and the reasoning beside the config.

```ts
// max sized min(db_max / instances, peak concurrency); acquire fails fast not forever (root rule 9); idle reclaimed
export const pool = new Pool({max: 10, connectionTimeoutMillis: 5_000, idleTimeoutMillis: 30_000}); // numbers documented
```

**Enforcement:** review; pool `max`, `connectionTimeoutMillis`, and idle/lifetime set explicitly with documented sizing; pool opened once and closed on shutdown (NODE-5), never per-request.

### 4.6 — Every list query is bounded.

**Reasoning, step by step:**
1. A query that returns a collection always carries a `LIMIT`. `select().from(users)` with no limit returns however many rows exist — fine on ten rows, an out-of-memory event and a stalled event loop on ten million. The bound is not optional and not a tuning step added later; an unbounded list is a latent incident the day the table grows, exactly the design-phase resource thinking [performance.md](../performance.md) demands.
2. Paginate with a cursor, not an offset. `OFFSET 10000 LIMIT 20` makes the database scan and discard 10,000 rows to return 20, and the cost grows with the page number; a cursor (`WHERE id > $lastId ORDER BY id LIMIT 20`) seeks straight to the next page in constant time and is stable when rows are inserted mid-iteration, which offset is not.
3. At the API surface this bounded, cursor-paged read is exposed as an `AsyncIterable<T>` whose generator pulls the next page only when the consumer drains the current one ([core 10.10](../typescript/10-api-design.md)) — the LIMIT here is the per-page fetch behind that iterator, so a caller who stops early never fetches the rest.

```ts
// cursor, not offset: WHERE id > $after seeks to the next page in constant time, stable under inserts; LIMIT always
const page = async (db: Db, after: string | undefined, size = 50) =>
  (await db.select().from(users).where(after ? gt(users.id, after) : undefined).orderBy(users.id).limit(size)).map(toUser);
```

**Enforcement:** review; every collection query has a `LIMIT`; cursor pagination over `OFFSET`; the API shape is `AsyncIterable<T>` ([core 10.10](../typescript/10-api-design.md)).

### 4.7 — N+1 is a bug, not a tuning opportunity.

**Reasoning, step by step:**
1. Issuing one query to fetch N parents and then one query per parent to fetch its children is N+1 round trips where one or two would do, and each round trip pays the network and pool cost the database is already the slowest resource to incur ([performance.md](../performance.md), batching). This guide takes that port verbatim: N+1 is a **bug**, classified and fixed like any other defect, not a performance nicety deferred to "later."
2. Fix it by batching. Collect the parent ids and fetch all children in one `inArray` query, then group in memory; or express the whole thing as a single `join`. The shape is the batching rule made concrete: one query with N ids, never N queries with one id each.
3. The query count of a request is part of review. A handler that loops over a result issuing a query per iteration is rejected the way a handler that swallows an error is rejected — visibly, at the boundary. Counting queries (in a test, or via the driver's logging) is how the bug is caught before it ships under load.

```ts
for (const order of orders) order.lines = await findLines(db, order.id); // bad — N+1: one query per order; the row mutation is a second smell (core 3.10)
// good — one batched query (inArray), grouped in memory; or a single join
const byOrder = Map.groupBy(await findLinesForOrders(db, orders.map((o) => o.id)), (l) => l.orderId);
```

**Enforcement:** review counts the queries a request issues; per-iteration queries in a loop are a bug; batch with `inArray`/joins.

### 4.8 — Rows are parsed at the repository boundary.

**Reasoning, step by step:**
1. A database row is external input. The schema can drift from the code, a migration can land ahead of a deploy, a column can be nullable in ways the types forgot — so a row arrives as `unknown` and is proven, not assumed, exactly as NODE-3 proves every other boundary. The driver's row type is a claim about the shape; the parse is the guarantee.
2. So map the row to a domain type at the repository boundary, before it crosses inward — a zod schema (`UserRow.parse(row)`) or an explicit mapper — and derive the domain type with `z.infer` so the parser and the type cannot drift ([core 10.7](../typescript/10-api-design.md)). Past the mapper the interior holds a validated value and never re-checks; the repository is the one place that touches a raw row.
3. Entities never leak past that boundary outward, either. The row shape — snake_case columns, driver date objects, join artifacts — stops at the repository; what travels inward is the domain type, and what reaches the API surface is the response DTO ([core 06](../typescript/06-classes-and-data-modeling.md)), never the row. This is parity with the [Kotlin entity/DTO rule](../kotlin-jvm/04-persistence.md): persistence shape and wire shape are decoupled, mapped explicitly, evolving independently.

```ts
const UserRow = z.object({id: z.uuid(), email: z.email(), credits: z.number().int()});
export type User = z.infer<typeof UserRow>;               // type follows the parser — cannot drift
const toUser = (row: unknown): User => UserRow.parse(row); // row is unknown until proven (NODE-3); never re-checked inside
```

**Enforcement:** review; rows mapped to domain types at the repository via zod/`z.infer` or an explicit mapper; row shapes never cross inward or reach the API surface (parity with [Kotlin 04](../kotlin-jvm/04-persistence.md)).

### 4.9 — Database errors wrap into domain errors at the repository.

**Reasoning, step by step:**
1. A driver error must not surface above the repository. A `pg` error with `code: '23505'` is the data layer's vocabulary; the domain and the API do not know what a SQLSTATE is. So the repository catches the driver fault and throws a typed domain error in its place — the boundary-wrapping the core guide mandates ([core 8.5](../typescript/08-error-handling.md)), applied to the persistence edge.
2. Translate the meaningful constraints into named domain failures. A unique-violation on `users.email` becomes an `EmailTakenError`, a foreign-key violation the typed failure the caller can act on — not a generic `Error`, not the raw driver error. These are operational errors the caller handles ([core 8.7](../typescript/08-error-handling.md)); the typed hierarchy makes the handler a `switch`, not a string-match on a SQLSTATE.
3. The `cause` chain stays intact and the SQL details stay below the boundary. The domain error wraps the driver error as its `cause` (`{ cause: toError(e) }`, [core 8.2](../typescript/08-error-handling.md)), preserving the original for the log trail, while the message and type the caller sees carry no table names, no query text, no SQLSTATE — only the domain meaning. The outermost boundary logs the full chain once (NODE-1, [core 8.5](../typescript/08-error-handling.md)).

```ts
try { return toUser((await db.insert(users).values(input).returning())[0]); } catch (e: unknown) {
  // unique_violation → typed domain failure (EmailTakenError extends RepositoryError); cause intact, SQL never leaked
  if (e instanceof DatabaseError && e.code === '23505') throw new EmailTakenError(input.email, {cause: toError(e)});
  throw new RepositoryError('failed to insert user', {cause: toError(e)});
}
```

**Enforcement:** review; driver errors caught at the repository and rethrown as typed domain errors with `cause` intact ([core 8.2](../typescript/08-error-handling.md), [8.5](../typescript/08-error-handling.md)); constraint violations become named failures; no SQLSTATE/SQL text above the boundary.

## Cross-references

- Boundary parsing, `z.infer` as the single source of type truth, cursor results as `AsyncIterable<T>`: [core 10.7 and 10.10](../typescript/10-api-design.md). Wrap-at-boundaries, log-once, `cause` chains, programmer-vs-operational errors: [core 8.2, 8.5, 8.7](../typescript/08-error-handling.md). Repositories as data + free functions, entity/DTO decoupling, lifecycle classes: [core 06](../typescript/06-classes-and-data-modeling.md).
- N+1, batching, and pool sizing as design-phase resource decisions: [performance.md](../performance.md). Entity/DTO discipline, forward-only migrations, HikariCP-style pool sizing on the JVM: [kotlin-jvm 04](../kotlin-jvm/04-persistence.md).
- Every-edge parsing, crash-only process boundary, and the pool as a named lifecycle opened at boot and drained on `SIGTERM`: NODE-3, NODE-1, NODE-5 ([README](./README.md)).
