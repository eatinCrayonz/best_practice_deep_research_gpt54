# Production Patterns for Application Code and Relational Databases

## Executive summary

The safest general pattern for application code in production is to treat the relational database as an explicit integration boundary, keep SQL understandable, and choose the thinnest abstraction that still buys enough productivity for the team. In practice, that usually means a typed query builder as the default center of gravity, raw SQL for the hottest or most database-specific paths, and a full ORM when CRUD speed, generated types, and team-wide conventions matter more than exact SQL control. Because the database vendor is unspecified, engine-specific behavior must be verified before shipping; even mainstream tooling documents that isolation behavior is not portable across vendors, and the examples below use PostgreSQL where a concrete engine example is unavoidable because raw `pg` is PostgreSQL-specific. citeturn11view1turn8view0turn9view0turn10view0turn5search12turn3view12

For schema change, the production default should be reviewed migrations executed by CI/CD, not ad hoc schema pushes from developer laptops. Zero-downtime change is fundamentally an expand-contract problem: make the schema additive first, deploy code that can tolerate both old and new shapes, backfill in chunks, flip reads, stop dual writes, and only then remove the old structure. Direct diff-to-database “push” workflows can be appropriate for prototyping, but generated or hand-authored SQL artifacts are materially easier to review, audit, stage, and coordinate under load. Blue-green and rolling application deploys help with cutover, but they do not eliminate the requirement that both versions remain schema-compatible while traffic drains and instances overlap. citeturn3view6turn3view7turn9view7turn9view8turn3view13turn4view4turn18search1

Operationally, most production failures come less from the choice of ORM than from pool exhaustion, long-running transactions, N+1 query patterns, and unobservable slow SQL. Reuse a small per-process client pool, leave connection headroom, add an external pooler or proxy in bursty/serverless systems, keep transactions short and bound to a single client/connection, retry only transient concurrency failures with bounded backoff, batch reads aggressively, test against a real ephemeral database in CI, and instrument latency, error rate, and normalized statement fingerprints. citeturn3view0turn6view0turn9view4turn6view1turn4view6turn4view10turn17search0turn3view14turn15view0turn14search1turn12search1turn12search0

Confidence is high for the general operating model and moderate for vendor-specific DDL and locking behavior because the database vendor is unspecified.

## Decision framework

For most production systems, the real choice is not “ORM or no ORM,” but how much abstraction to buy at each layer. The database boundary has three competing goals: productivity, control, and operational predictability. Thin abstractions preserve the database as a first-class design surface; thicker abstractions reduce boilerplate and standardize common work, but become leaky on complex queries and concurrency-sensitive workflows. The relative performance profile below is an architectural expectation, not a universal benchmark ranking. Exact cross-library performance rank order is **UNKNOWN** without workload-specific measurement. citeturn3view1turn11view1turn8view0turn6view3turn4view0

| Access style | Productivity | Runtime profile | Control | Type safety | Maintainability | Learning curve | Best fit | Evidence basis |
|---|---|---|---|---|---|---|---|---|
| Raw SQL | Lower initial velocity, but maximal precision | Lowest library overhead; performance depends mostly on SQL quality | Highest | Manual unless paired with codegen/validators | Excellent if SQL is centralized and reviewed; poor if scattered through handlers | Highest | Hot paths, complex analytical SQL, vendor-specific features, teams fluent in SQL | citeturn3view1turn3view0turn6view0 |
| Query builder | Strong balance of speed and explicitness | Low overhead; SQL remains visible and predictable | High | Usually strongest compile-time query typing | Often the best long-run balance | Medium | Greenfield services, SQL-first teams, most production backends | citeturn11view1turn3view9turn8view0 |
| Full ORM | Highest for conventional CRUD and relation traversal | More runtime behavior and more chances for hidden query shape issues | Medium to low | Strong model/query typing in modern tools | Good until query patterns become atypical or relation loading becomes implicit | Medium | CRUD-heavy product teams, multi-developer consistency, batteries-included workflows | citeturn9view0turn3view3turn6view3turn10view3 |

The strongest default recommendation for many production services is therefore: **use a typed query builder unless there is a clear reason to move up or down the abstraction stack**. Move downward to raw SQL when SQL itself is the product of the service. Move upward to a full ORM when the dominant engineering cost is repetitive CRUD and relation handling rather than designing specialized SQL. citeturn11view1turn8view0turn9view0turn10view3

## Migrations and schema evolution

A practical production hierarchy is: reviewed SQL migrations first, generated-and-reviewed SQL second, direct schema push last. Drizzle explicitly offers both ends of that spectrum: `drizzle-kit push` applies diffs directly to the database and is positioned as a code-first approach well suited to rapid prototyping, while `drizzle-kit generate` plus `drizzle-kit migrate` produces SQL artifacts you can review and apply. Prisma’s production guidance is stricter: use `migrate deploy` in automated CI/CD, not local manual deployment to production. TypeORM exposes migration generation too, but separately warns that direct schema synchronization (`synchronize: true`) is unsafe once production data exists. citeturn3view6turn3view7turn9view7turn9view8turn6view2turn13search12turn10view3

The most robust production stance is **forward-safe by default, reversible only when genuinely safe**. Kysely gives you explicit up/down migration primitives, and that is valuable in development and pre-production. But once a migration includes lossy data transformation, backfills, dropped columns, or changed semantics, “reversible” becomes more about rollout design than about a `down()` function. In real production systems, safety more often comes from forward-only additive steps plus expand-contract than from pretending every live data mutation can be rolled back symmetrically. citeturn11view1turn3view9turn3view13turn18search1

The expand-contract sequence is the correct zero-downtime mental model for breaking schema changes. PlanetScale’s guidance is explicit: expand schema, dual-write, migrate data, cut reads, stop writing old shape, then contract. Prisma’s current production migration guidance and Data Guide say the same thing in slightly different words, including additive deployment, migration/backfill, staged read cutover, and careful testing before old structures are removed. This is also why blue-green app deploys do not replace expand-contract: if old and new application binaries overlap against one database, every intermediate schema state still needs to be backward compatible. citeturn3view13turn4view4turn18search1

```mermaid
flowchart LR
    A[Expand schema<br/>add new nullable columns or tables] --> B[Deploy app versions that tolerate old and new shapes]
    B --> C[Dual-write old and new structure]
    C --> D[Backfill historical rows in chunks]
    D --> E[Validate correctness and performance]
    E --> F[Cut reads to new structure]
    F --> G[Stop writes to old structure]
    G --> H[Contract schema<br/>drop old columns or tables]
    B -. rolling deploy / blue-green overlap .-> C
```

For evolution under load, backfills should be treated as background jobs, not giant one-shot migrations. Chunk them, checkpoint progress, throttle them, and keep each chunk transaction short. If application-level dual-write is impractical during rolling deploys, database triggers are a vendor-specific fallback for temporary synchronization because trigger functions can automatically run on data changes. That said, triggers should usually be temporary and explicit, because hidden write logic in the database makes incident response, replay, and application reasoning harder. citeturn18search1turn18search2turn18search7turn4view10

When the engine or platform offers online schema change, use it as an implementation of the **expand** step, not as a substitute for backward compatibility. The reason is simple: the operational danger is not only lock time, but also application-version overlap and semantic compatibility. PostgreSQL’s docs are a useful cautionary example: DDL and `AccessExclusiveLocks` can conflict with standby queries, and long-running queries can delay WAL replay or get canceled. On any vendor, check the exact lock behavior of the specific ALTER you plan to run before treating it as “online enough.” citeturn7view0turn3view13

## Pooling, transactions, and query shape

Connection pooling has two distinct layers. The first is the **client-side pool** inside the application process. In node-postgres, pools are lazy, the default `idleTimeoutMillis` is 10 seconds, and the pool sizing guide recommends leaving headroom rather than allocating the entire server connection budget to application instances. Prisma documents the same basic reality from another angle: pool size and idle-time behavior exist at the client layer too, and external poolers become important when many functions or processes each carry their own local pool. The main production rule is simple: one long-lived client pool per process, reused across requests; never create a new database client object per request. citeturn3view0turn6view0turn4view2turn9view4

The second layer is the **server-side pooler/proxy**, such as PgBouncer or a managed connection proxy. This layer is often necessary in autoscaling and serverless topologies because the database can only support so many backend connections, while application process count can spike hard. PgBouncer’s feature matrix is especially important operationally: transaction pooling deliberately breaks some session-level assumptions, including `SET/RESET`, `LISTEN`, holdable cursors, temp-table behaviors, and session advisory locks. That makes transaction pooling highly effective, but only when the application cooperates with those limits. citeturn6view1turn6view0turn9view4

Prepared statements are where pooling choices become concrete. In node-postgres, named prepared statements cache the query plan **per connection**, not globally. That is excellent when connections are reused predictably, but it means behavior changes when a server-side pooler reassigns transactions across backend connections. PgBouncer’s current docs say protocol-level prepared plans can work in transaction pooling if `max_prepared_statements` is enabled, but old assumptions based on session affinity still need to be validated carefully. Session pooling remains the least surprising option when prepared-statement semantics matter more than aggressive multiplexing. citeturn3view1turn6view1turn5search13

Transaction boundaries should follow **one invariant, one connection, minimal time**. In raw `pg`, all statements inside a transaction must use the same client instance, and the docs explicitly warn not to use `pool.query` for transactional work. Prisma’s interactive transaction docs say the quiet part out loud: keep them short, do not perform network requests inside them, and get in and out quickly because long transactions hurt performance and can deadlock. TypeORM makes the same point in a different form: inside a transaction, every operation must go through the transactional entity manager or query runner that owns the single underlying connection. citeturn4view6turn4view9turn4view10turn10view0

Isolation should be chosen by anomaly budget, not by habit. TypeORM’s current transaction docs correctly point out that isolation implementations are not agnostic across databases. PostgreSQL is still a helpful reference model: Read Committed prevents dirty reads but still allows nonrepeatable reads and serialization anomalies, while Serializable eliminates the anomaly classes shown in the current docs. The practical conclusion is to start narrower than Serializable when contention is high and invariants are local, but use Serializable when the correctness requirement truly demands it. Then be prepared to retry. citeturn10view0turn3view12

Retry logic belongs around the **whole transaction function**, not around individual statements. PostgreSQL’s current serialization failure guidance is explicit: applications at Repeatable Read and Serializable must be prepared to retry `40001` serialization failures, and retrying `40P01` deadlock failures is often advisable too. Prisma documents the same operational idea through its P2034 guidance. Bounded retries, exponential backoff, and jitter are usually enough; unbounded blind retries can turn a contention spike into a self-inflicted denial of service. citeturn17search0turn3view3turn4view11

```mermaid
flowchart LR
    A[Request enters service] --> B[Validate input and auth]
    B --> C[Do non-transactional work outside the DB transaction]
    C --> D[Acquire pooled connection]
    D --> E[BEGIN]
    E --> F[Read or lock rows needed for the invariant]
    F --> G[Write rows]
    G --> H[COMMIT]
    H --> I[Run external side effects after commit]
    I --> J[Release connection]
    E --> K[ROLLBACK on error]
    K --> J
```

N+1 prevention is one of the few rules that applies equally across raw SQL, query builders, and ORMs. GraphQL’s performance guide describes the generic shape: one initial fetch followed by one query per result row. The generic remedies are equally stable: use joins when the row explosion is acceptable, batch by foreign key with `IN`/`ANY` when joins would inflate result sets or duplicate data, and use request-scoped loaders when the call graph is resolver-like. Prisma gives concrete patterns for all three—`include`, `in`, `relationLoadStrategy: "join"`, and same-tick batching of `findUnique()` calls—while TypeORM recommends `leftJoinAndSelect` and warns that lazy loading can create N+1 if used carelessly. citeturn3view14turn4view0turn4view1turn6view3

Long-running transactions deserve special suspicion. On PostgreSQL, very long-lived write transactions can delay standby readiness, and long-running standby queries can be canceled or can delay WAL replay, which is a concrete reminder that “it’s only a read transaction” does not mean “it’s operationally free.” That matters for batch jobs, backfills, admin scripts, and migration tooling as much as for request-path code. citeturn7view0

## Type safety, testing, and observability

There are three durable ways to keep schema and application types aligned. The first is **generated types from schema**, exemplified by Prisma, whose client docs describe full query type safety and generated input/select/include types derived from the data model. The second is **schema-in-TypeScript as source of truth**, exemplified by Drizzle, which uses the TS schema for both query construction and migration generation or push workflows. The third is **database-typed query construction**, exemplified by Kysely, which emphasizes compile-time query typing and can use database-driven code generation so the database becomes the source of types. TypeORM gives good typing around entities and repositories, but its type-safety model is more entity-centric than exhaustively query-shape-centric. Raw `pg` gives no schema-derived types out of the box, so teams should add deliberate row mapping and runtime boundary validation if they choose it. citeturn9view0turn9view1turn3view6turn8view0turn11view1turn10view4turn3view1

Testing should separate **logic isolation** from **database truth**. For unit tests, Prisma’s current testing docs recommend mocking Prisma Client and show both singleton and dependency-injection approaches, which generalizes well even if the project uses another library: mock the narrow data-access interface, not the entire database. But mock-heavy testing cannot validate actual SQL, indexes, transaction semantics, or migration effects. Those need integration tests against a real relational engine. Prisma’s integration testing guide shows the canonical flow clearly: start an isolated database environment, apply migrations, run tests, then destroy the environment. citeturn16view0turn15view0

Ephemeral databases are now the preferred way to do that in CI because they maximize realism without contaminating shared environments. Testcontainers for Node describes its model as lightweight, throwaway instances of common databases for tests, and its quickstart/global-setup docs note that teams can either reuse a container across a suite or start per-test containers when parallelism and isolation matter more. The correct choice is workload-dependent, but the principle is stable: the closer your test environment is to real database behavior, the more confidence you should place in the result. citeturn14search1turn14search3

Observability should begin with normalized SQL visibility, not just ORM-level logs. PostgreSQL’s `pg_stat_statements` tracks planning and execution statistics for SQL statements, and PostgreSQL can compute query identifiers that align log lines, activity views, and statement statistics. On the client side, Prisma supports stdout and event-based query logging, while TypeORM can log query, error, and schema events. For cross-service tracing and metrics, OpenTelemetry’s database semantic conventions provide a standard vocabulary for spans, metrics, and logs, which matters once more than one service, framework, or language hits the same database. citeturn12search1turn12search5turn12search2turn12search15turn12search0turn12search16

The practical production bundle is therefore: slow-query statistics at the database, sampled or event-based query logging in the application, latency/error dashboards, and distributed traces that put the database span beside the HTTP request or background job that caused it. That combination is far more useful than any single layer by itself. citeturn12search1turn12search2turn12search15turn12search0

## TypeScript and JavaScript tool survey

Absolute runtime rank order across Prisma, Drizzle, Kysely, TypeORM, and raw `pg` is **UNKNOWN** in the abstract because it varies with query shape, network latency, row mapping cost, relation-loading strategy, and the amount of SQL emitted per logical operation. The table below therefore summarizes **relative architectural characteristics**, not benchmark scores. “Maturity” is an assessment based on documentation breadth, production features, and visible maintenance signals in official materials. citeturn3view1turn11view1turn8view0turn6view3turn9view0

| Tool | Category | Maturity and support | Type safety | Migration tooling | Runtime profile | Ergonomics | Typical use-cases | Main limitations | Right choice when | Sources |
|---|---|---|---|---|---|---|---|---|---|---|
| raw `pg` | Driver + raw SQL | High operational maturity assessment; minimal surface, current pooling/transactions/prepared-statement docs | Manual row typing only; no schema-derived model layer | Manual or external migration tool | Lowest abstraction overhead; maximal SQL control | Verbose; you own SQL, mapping, and conventions | Hot paths, complex SQL, vendor-specific features, DBA-friendly services | More boilerplate; type drift risk if mapping is sloppy; PostgreSQL-specific | SQL is the product, or control and auditability outweigh convenience | citeturn3view1turn3view0turn4view6turn6view0 |
| Prisma | Full ORM with generated client | High; broad official docs across type safety, transactions, migrations, testing, logging, deployment | Very strong generated query/input/output types | Generated SQL migrations, `migrate deploy`, expand-contract guidance | Excellent for CRUD and team consistency; exact perf depends on emitted SQL and relation strategy | Best-in-class DX for many teams | SaaS CRUD, multi-team apps, standardized service development | Less direct SQL control; N+1 requires conscious API choices; migrations are SQL-file driven, not TS-first | Productivity, generated types, and cohesive tooling matter most | citeturn9view0turn3view3turn4view0turn9view7turn12search2turn15view0turn16view0 |
| Drizzle | SQL-like ORM / query builder | Medium-high and rapidly evolving; active 2024–2025 release cadence and explicit community channels | Strong TS from schema-in-code; SQL-like API | Direct `push`, generated SQL, runtime/external application options | Thin and driver-native; docs emphasize one-query relational API and zero dependencies | Very good if the team already thinks in SQL | Greenfield services, SQL-first teams, serverless-friendly stacks | Direct `push` is less auditable for serious prod change control; ecosystem still moving quickly | You want SQL feel, strong typing, and light runtime abstraction | citeturn8view0turn3view6turn3view7turn8view1 |
| Kysely | Type-safe query builder | Medium-high; focused, community-driven, thin abstraction with optional migration/runtime tools | Excellent compile-time query typing; database can be source of types via codegen | Optional up/down primitives and CI-friendly migration execution | Very low abstraction overhead due to 1:1 SQL compilation | Excellent for SQL-proficient teams; less magic than ORM-heavy tools | Services with complex SQL and disciplined data-access design | Fewer batteries for relations, schema workflow, and ecosystem integrations than Prisma | You want the query-builder default with strong static guarantees | citeturn11view1turn3view9turn11view0 |
| TypeORM | Full ORM + query builder | High feature breadth, stable v0.3, active maintainers/community surface | Good entity/model typing; query typing is less exact than Kysely or Prisma-generated types | Automatic generation + migrations; warns against production schema sync | Heavier entity/repository layer; query builder/raw paths are important for hotspots | Familiar ORM model; supports both DataMapper and ActiveRecord | Existing enterprise apps, decorator-heavy domains, teams already invested in entity-centric modeling | Lazy loading can create N+1; abstraction can become leaky; `synchronize: true` unsafe in production | You already use it successfully, or specifically want its ORM patterns and broad DB support | citeturn10view3turn10view4turn6view5turn6view3turn13search12 |

Two patterns stand out. First, the center of gravity in current TypeScript database tooling has shifted toward **SQL-first type safety**: Drizzle and Kysely both preserve database concepts more directly than classic ORMs do. Second, full ORMs still remain attractive when the dominant engineering cost is repetitive CRUD and relational object handling rather than designing specialized SQL. Prisma is the strongest batteries-included option in that category today, while TypeORM remains more attractive in existing codebases than as the obvious new default. citeturn8view0turn11view1turn9view0turn10view3turn6view3

## Code patterns and diagrams

The following examples use PostgreSQL syntax where needed because raw `pg` is PostgreSQL-specific, and the retry SQLSTATE values are PostgreSQL values from current concurrency docs. The patterns themselves generalize to other relational engines. The two core ideas from the official docs are: use a **single client** for a raw transaction, and keep transaction scopes **short and callback-bound** in higher-level libraries. citeturn4view6turn11view0turn17search0

```ts
// raw-pg-transaction.ts
import { Pool } from "pg";

const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 10_000,
  connectionTimeoutMillis: 2_000,
});

const RETRYABLE = new Set(["40001", "40P01"]); // PostgreSQL codes

function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function safeRollback(client: { query: (sql: string) => Promise<unknown> }) {
  try {
    await client.query("ROLLBACK");
  } catch {
    // Ignore rollback failures; the connection will be released.
  }
}

export async function transferFunds(
  fromAccountId: number,
  toAccountId: number,
  amount: number
): Promise<void> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const client = await pool.connect();

    try {
      await client.query("BEGIN ISOLATION LEVEL SERIALIZABLE");

      const fromResult = await client.query<{ balance: string }>(
        "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
        [fromAccountId]
      );

      if (fromResult.rowCount !== 1) {
        throw new Error("source account not found");
      }

      const balance = Number(fromResult.rows[0].balance);
      if (balance < amount) {
        throw new Error("insufficient funds");
      }

      await client.query(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        [amount, fromAccountId]
      );
      await client.query(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        [amount, toAccountId]
      );

      await client.query(
        `INSERT INTO ledger_entries (from_account_id, to_account_id, amount)
         VALUES ($1, $2, $3)`,
        [fromAccountId, toAccountId, amount]
      );

      await client.query("COMMIT");
      return;
    } catch (err: any) {
      await safeRollback(client);

      if (RETRYABLE.has(err?.code) && attempt < 4) {
        const backoffMs = Math.min(50 * 2 ** attempt, 1_000) + Math.random() * 50;
        await sleep(backoffMs);
        continue;
      }

      throw err;
    } finally {
      client.release();
    }
  }

  throw new Error("transaction failed after retries");
}
```

For batching, the most reliable pattern is to turn many point-lookups into one set-based query and then regroup in memory. That is the same idea behind DataLoader and behind Prisma’s documented `in`/join strategies. citeturn3view14turn4view0

```ts
// raw-pg-batching.ts
import type { Pool } from "pg";

type PostRow = {
  author_id: number;
  id: number;
  title: string;
};

export async function loadPostsByAuthorIds(
  pool: Pool,
  authorIds: readonly number[]
): Promise<PostRow[][]> {
  const deduped = [...new Set(authorIds)];
  if (deduped.length === 0) return [];

  const { rows } = await pool.query<PostRow>(
    `
      SELECT author_id, id, title
      FROM posts
      WHERE author_id = ANY($1::int[])
      ORDER BY author_id, id
    `,
    [deduped]
  );

  const grouped = new Map<number, PostRow[]>();
  for (const id of deduped) grouped.set(id, []);
  for (const row of rows) grouped.get(row.author_id)!.push(row);

  return authorIds.map((id) => grouped.get(id) ?? []);
}
```

Kysely’s transaction style is a good illustration of the “typed query builder” middle ground: explicit SQL structure, compile-time typing, but much less stringly-typed code than raw SQL. citeturn11view0turn11view1

```ts
// kysely-transaction-and-batching.ts
import { Kysely } from "kysely";

interface DB {
  orders: {
    id: number;
    customer_id: number;
    status: "pending" | "paid";
  };
  order_items: {
    id: number;
    order_id: number;
    sku: string;
    qty: number;
  };
  posts: {
    id: number;
    author_id: number;
    title: string;
  };
}

export async function createOrder(
  db: Kysely<DB>,
  customerId: number,
  items: Array<{ sku: string; qty: number }>
): Promise<number> {
  return db.transaction().execute(async (trx) => {
    const order = await trx
      .insertInto("orders")
      .values({
        customer_id: customerId,
        status: "pending",
      })
      .returning("id")
      .executeTakeFirstOrThrow();

    await trx
      .insertInto("order_items")
      .values(
        items.map((item) => ({
          order_id: order.id,
          sku: item.sku,
          qty: item.qty,
        }))
      )
      .execute();

    return order.id;
  });
}

export async function loadPostsByAuthorIds(
  db: Kysely<DB>,
  authorIds: readonly number[]
) {
  if (authorIds.length === 0) return new Map<number, DB["posts"][]>();

  const rows = await db
    .selectFrom("posts")
    .select(["author_id", "id", "title"])
    .where("author_id", "in", [...new Set(authorIds)])
    .orderBy("author_id")
    .orderBy("id")
    .execute();

  const grouped = new Map<number, DB["posts"][]>();
  for (const id of authorIds) grouped.set(id, []);
  for (const row of rows) grouped.get(row.author_id)!.push(row);

  return grouped;
}
```

For migrations, tools that support explicit `up`/`down` files are useful, but for production rollouts the more important property is that the **up** path is additive and backward compatible. citeturn3view9turn3view13

```ts
// kysely-migration.ts
import { Kysely, sql } from "kysely";

// Expand step: add the new column, but do not remove the old one yet.
export async function up(db: Kysely<unknown>): Promise<void> {
  await db.schema
    .alterTable("users")
    .addColumn("status_v2", "text")
    .execute();

  await sql`
    comment on column users.status_v2 is
    'expand step; deploy dual-write before backfill and cutover'
  `.execute(db);
}

// Fine for dev/pre-prod. In production, many teams prefer explicit roll-forward
// after any live data migration rather than relying on a generic down().
export async function down(db: Kysely<unknown>): Promise<void> {
  await db.schema
    .alterTable("users")
    .dropColumn("status_v2")
    .execute();
}
```

## Recommendations by scenario

For a new production service with engineers who are comfortable reading SQL, the strongest default recommendation is **Kysely or Drizzle, plus reviewed migrations, plus real-database integration tests**. That combination keeps SQL visible, preserves strong type safety, and avoids the heaviest ORM abstraction costs without falling back to pure hand-written driver code everywhere. citeturn11view1turn8view0turn3view6turn15view0

For a product team whose dominant workload is CRUD, relation traversal, and rapid feature delivery across several developers, **Prisma is usually the strongest choice**. Its advantage is not that it magically makes SQL faster, but that it unifies type generation, transactions, production-safe migration deployment, testing guidance, and logging into one operating model. That tends to reduce organizational mistakes even when the underlying SQL still needs care. citeturn9view0turn3view3turn9view7turn12search2turn16view0turn15view0

For services where the database is a performance-critical subsystem, where vendor-specific SQL matters, or where complex analytical statements are common, **raw `pg` or raw SQL embedded in a thin typed query layer** is the right posture. The cost is more boilerplate and more responsibility for schema/type discipline. The payoff is control over text, plans, retries, locks, batching, and migration shape. citeturn3view1turn4view6turn6view0turn17search0

For existing TypeORM codebases, the economically rational move is usually **not** “rewrite to the trendy tool,” but rather “tighten transactional discipline, avoid lazy-loading surprises, use QueryBuilder or raw SQL for hotspots, and enforce migrations instead of `synchronize`.” TypeORM remains viable, but it benefits more than the newer SQL-first tools from deliberate guardrails. citeturn6view3turn10view0turn13search12

The most important conclusion is architectural rather than library-specific: production correctness is won by **backward-compatible schema design, small pools, short transactions, set-based queries, real integration tests, and good observability**. Once those disciplines are in place, the tool choice matters; before those disciplines are in place, the tool choice is often a distraction. citeturn3view13turn6view0turn17search0turn3view14turn15view0turn12search1