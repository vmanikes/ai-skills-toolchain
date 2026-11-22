---
name: db-design
description: Designs and reviews database tables, columns, indexes, queries and migrations for relational (Postgres, MySQL, SQLite, SQL Server) and NoSQL (DynamoDB, MongoDB, Cassandra) databases. Enforces standard id/created_at/updated_at columns with the right id type per engine, flags enums and risky foreign keys, orders composite indexes by ESR (Equality, Sort, Range), audits redundant or missing indexes against the codebase's query patterns, and writes safe online migrations in the repo's own migration tool. Use when the user says "design a table", "create a schema", "add a column", "add an index", "optimize indexes", "review this migration", "why is this query slow", or "data model for DynamoDB".
license: MIT
---

# DB Design

Designs new tables and reviews existing schemas, indexes, queries and migrations. It proposes concrete changes, explains the cost behind every recommendation, and writes the ones the user approves in the repo's own migration or ORM format.

**Golden rule:** follow the repository's existing database conventions. Detect the engine, the migration/ORM tool and the naming style before suggesting anything, and write output in that tool's format. Never introduce a new migration tool or ORM. If the engine can't be determined, ask.

## Modes

Pick the mode from the request, then run only the phases it lists. Always start with Phase 0.

| Mode | Triggered by | Phases |
|------|--------------|--------|
| **Design** | "design a table", "create a schema", "data model for..." | 0, 1, 2, 3, 5, 6 |
| **Review** | "review this migration/schema", a migration file in the diff | 0, 1, 2, 3, 5, 6 |
| **Optimize indexes** | "optimize indexes", "do we need this index", "missing indexes" | 0, 3, 6 |
| **Query review** | "why is this query slow", "review this query", "EXPLAIN this" | 0, 4, (3 if an index would fix it), 6 |

After each phase that proposes changes, **stop and ask the user which suggestions to apply** before editing files.

## Phase 0. Context

Collect this before designing anything. See [REFERENCE.md](REFERENCE.md#convention-discovery) for where to look.

| Item | What to find |
|------|--------------|
| Engine and version | Postgres 16, MySQL 8, DynamoDB... from docker-compose, connection strings, ORM config, driver dependencies. Features depend on version (e.g. `uuidv7()` is Postgres 18+). |
| Migration / ORM tool | Prisma, Django, Alembic, SQLAlchemy, ActiveRecord, TypeORM, Drizzle, Ent, golang-migrate, goose, Flyway, Liquibase, raw SQL files |
| Naming conventions | Table names (plural/singular, snake_case), column names, index and constraint names, with a file:line example of each |
| Existing standard columns | How existing tables declare `id`, timestamps, soft delete, tenant ID. Reuse that pattern. |
| Scale and workload | Expected row count and growth, read/write ratio, peak write throughput. Ask if unknown; it changes FK, index and migration advice. |
| Access patterns | The queries this table must serve: filters, sort order, pagination, lookups by key. **Required for NoSQL**, where the design follows the access patterns. |

Show the user a short summary of what was found. Say "none found" rather than guessing.

## Phase 1. Tables

### Relational tables

Every table gets these columns unless an exception below applies:

| Column | Type | Notes |
|--------|------|-------|
| `id` | Primary key with **monotonically increasing** values: UUIDv7, ULID, or unsigned `BIGINT` identity | Sequential keys append to the end of the B-tree instead of splitting random pages. **Random UUIDv4 is not monotonic**: it fragments the primary key index and slows inserts on large tables. If the repo uses UUIDv4, point this out once. See [REFERENCE.md](REFERENCE.md#id-types-by-engine) for the generator per engine. |
| `created_at` | Timezone-aware timestamp (`timestamptz`, `DATETIME(6)` stored in UTC), `NOT NULL DEFAULT now()` | Set once on insert. |
| `updated_at` | Same type, `NOT NULL DEFAULT now()` | Most engines **don't update it automatically**. Add a trigger (Postgres), `ON UPDATE CURRENT_TIMESTAMP` (MySQL), or the ORM's auto-update option. See [REFERENCE.md](REFERENCE.md#updated_at-mechanisms). |

**Exceptions** (propose them and let the user decide, don't silently skip columns):
- **Join tables** for many-to-many relations can use a composite primary key (`(user_id, group_id)`) instead of a surrogate `id`.
- **Append-only tables** (events, audit logs, ledger entries) never update rows, so `updated_at` adds nothing.

**Then ask the user whether they want ascending or descending indexes on `created_at` and `updated_at`**, explaining when each helps: a descending index serves "newest first" feeds; an index on `updated_at` serves sync/"changed since" queries. Don't add them by default; on write-heavy tables every index has a cost.

**Optional columns to offer** when the use case suggests them:
- `deleted_at` for soft delete, if records must be recoverable or auditable. Remind the user that unique constraints and indexes then usually need `WHERE deleted_at IS NULL`.
- `tenant_id` for multi-tenant apps, `NOT NULL`, leading column of tenant-scoped indexes.
- `version` integer for optimistic locking, if concurrent edits to the same row are likely.

### Non-relational tables

The same three fields should exist, **but choose the id type based on how the database partitions data**:

| Partitioning model | Engines | Id guidance |
|--------------------|---------|-------------|
| **Hash-partitioned** (key is hashed to pick a partition) | DynamoDB, Cassandra, MongoDB with a hashed shard key | Any high-cardinality id spreads evenly, including ULID. Hot partitions come from **low-cardinality or skewed keys** (a partition or GSI key of `status`, `date`, `country`), or one key getting most of the traffic. |
| **Range-partitioned** (rows stored in key order, split into ranges) | Spanner, CockroachDB, Bigtable, HBase, MongoDB with a ranged shard key | **Avoid monotonic ids as the leading key**: every insert lands on the last range and creates a hotspot. Use random UUIDv4, a hashed prefix, or a bit-reversed sequence. |

For **DynamoDB**, design keys from the access patterns gathered in Phase 0:
- Partition key: high cardinality, evenly accessed (e.g. `USER#<id>`).
- Sort key: supports the range and sort queries within a partition. A ULID or ISO timestamp works well here because it sorts by time.
- One GSI per additional access pattern; check each GSI key for the low-cardinality hot-partition problem.
- `created_at`/`updated_at` are attributes; index them only via a GSI that a real access pattern needs.

For **MongoDB**, `ObjectId` is already time-ordered and fine as `_id` unless the collection is sharded on a ranged `_id` key.

## Phase 2. Columns

### Data types

Check the type of every column the user provides:

| Problem | Recommendation |
|---------|----------------|
| Money in `FLOAT`/`DOUBLE`/`REAL` | `DECIMAL(19,4)` / `NUMERIC`, or integer minor units (cents) plus a currency column. Floats can't represent 0.10 exactly. |
| Timestamps without time zone, or stored as strings | Timezone-aware type stored in UTC |
| Nullable by default | `NOT NULL` unless "unknown" is a real, distinct state. Nulls complicate queries, uniqueness and indexes. |
| `VARCHAR(255)` everywhere | Postgres: `text` with a `CHECK (length(...) <= n)` if a limit matters. MySQL: size to the real maximum; it affects index key length and memory temp tables. |
| JSON column for structured, queried data | Real columns for fields that are filtered, joined or constrained. JSON only for truly variable or opaque data. |
| Booleans as `CHAR(1)`/strings | Native boolean (or `TINYINT(1)` in MySQL) |
| No uniqueness on natural keys | `UNIQUE` constraint on things that must be unique (email per tenant, external IDs, slugs). The app can't enforce uniqueness under concurrency; the database can. |

### Enums

If the user uses an `ENUM` type, suggest not doing it and encourage one or both of:

1. **Lookup table**: `order_statuses (id, code, label)` referenced from the main table. Values can be added, renamed or retired with a row change, and can carry metadata (display label, sort order, is_active).
2. **CHECK constraint**, if the engine supports it (Postgres, SQL Server, SQLite, MySQL 8.0.16+): `status text NOT NULL CHECK (status IN ('pending','paid','shipped'))`. Changing the list is a quick constraint swap.

Explain why: adding a value to a native enum needs a migration (in MySQL, often a full table rebuild), removing or renaming a value is hard or impossible in Postgres, and ordering is fixed by declaration order. See [REFERENCE.md](REFERENCE.md#enum-alternatives) for DDL. Small, truly fixed sets (e.g. a 2-value direction) are fine to leave; say so rather than insisting.

### Foreign keys

The case for foreign keys is **use with caution**. When the schema uses them:

1. **Check for `ON DELETE CASCADE`** and flag it. On large tables, deleting one parent row can delete or lock thousands of child rows in one long transaction, and a mistaken delete silently removes data across tables. Suggest `RESTRICT`/`NO ACTION` with explicit application deletes, or soft delete.
2. **Explain the cost on large or high-throughput tables**: every insert or update of the child must look up the parent to keep referential integrity, and parent deletes/updates must check the children. That adds reads and locks to hot write paths. Suggest avoiding FKs on these tables and enforcing integrity in the application (with a periodic orphan check) instead.
3. **Require an index on the referencing column** if the FK is kept. Postgres and SQL Server don't create one automatically, so every parent delete scans the child table. (MySQL InnoDB creates it.)
4. Note engine limits: sharded MySQL (Vitess, PlanetScale) restricts FKs; most NoSQL engines have none.

Present the tradeoff and let the user decide. On small, low-write tables FKs are usually worth keeping.

## Phase 3. Indexes

### ESR ordering for composite indexes

If the engine uses B-tree indexes (Postgres, MySQL/InnoDB, SQLite, SQL Server, Oracle, MongoDB), order composite index columns by **ESR**:

1. **Equality**: columns compared with `=` or `IN` (small list)
2. **Sort**: columns in `ORDER BY`
3. **Range**: columns compared with `>`, `<`, `BETWEEN`, `LIKE 'abc%'`

Example for `WHERE tenant_id = ? AND status = ? AND created_at > ? ORDER BY priority DESC`:
- ✅ `(tenant_id, status, priority DESC, created_at)`: equality narrows to one contiguous slice, rows are already sorted by priority, and the range filter is applied while reading.
- ❌ `(created_at, tenant_id, status)`: the range on the leading column means the index can't use the equality columns to narrow the scan, and the database must sort the result.

If the user isn't following ESR, suggest the reorder with the specific query it helps and why. More examples in [REFERENCE.md](REFERENCE.md#esr-examples).

For tenant-scoped tables, `tenant_id` is almost always the first equality column.

### Partial indexes

If a column being indexed contains many NULL values (or soft-deleted rows), suggest a partial index that ignores them, e.g. `CREATE INDEX ... ON orders (shipped_at) WHERE shipped_at IS NOT NULL` or `... WHERE deleted_at IS NULL`. The index is smaller, faster to scan and cheaper to maintain.

- Only suggest it if queries never need the excluded rows from this index (e.g. never filter on `IS NULL`), and the query's `WHERE` clause matches the index predicate.
- Engine support: Postgres and SQLite (`WHERE`), SQL Server (filtered index), MongoDB (`partialFilterExpression`). **MySQL has no partial indexes**; skip or suggest a functional index on a generated column. See [REFERENCE.md](REFERENCE.md#partial-index-support).

### Index cost audit

Every index slows writes and takes memory. In Review and Optimize modes, flag:

| Finding | Why it matters |
|---------|----------------|
| **Redundant index**: its columns are a leading prefix of another index (`(a)` when `(a, b)` exists) | The longer index already serves those queries |
| **Duplicate index**: same columns, same order | Pure write cost |
| **Unused index** | Write and storage cost with no benefit. Only claim "unused" with evidence: index usage stats (query in [REFERENCE.md](REFERENCE.md#index-audit-queries)) or no matching query in the codebase. Stats reset on restart and replicas track separately, so recommend confirming before dropping. |
| **Low-selectivity single-column index** (boolean, small status set) | The planner usually ignores it; better as a later column in a composite or as a partial index |
| **Covering opportunity** | A hot query that reads a few extra columns can be served from the index alone with `INCLUDE (...)` (Postgres 11+, SQL Server) or by appending the columns |
| **Too many indexes on a write-heavy table** | Each insert updates every index; call out the count and the write path it affects |

### Optimize indexes mode

When the user asks to optimize indexes:

1. Find the table's current indexes (migrations, schema dump, ORM model definitions).
2. Scan the codebase for every query pattern that touches the table: raw SQL, query builders, ORM calls (`filter`, `where`, `order_by`, `find`, repository methods). See [REFERENCE.md](REFERENCE.md#convention-discovery) for search patterns.
3. Map each query to the index that serves it:

```
| Query (file:line)                       | Filter / sort                         | Index used             | Status    |
|-----------------------------------------|---------------------------------------|------------------------|-----------|
| orders/repo.go:42 ListByCustomer        | customer_id =, ORDER BY created_at    | idx_orders_customer    | ⚠️ sort not covered |
| orders/repo.go:88 FindPendingOlderThan  | status =, created_at <                | none                   | ❌ missing |
| (no query found)                        | -                                     | idx_orders_coupon_code | 🗑 unused? |
```

4. Suggest indexes to create (ESR-ordered) and indexes to drop, each with the queries it affects.

## Phase 4. Queries

Review queries in the changeset or the ones the user points to:

| Anti-pattern | Fix |
|--------------|-----|
| Function or cast on an indexed column: `WHERE lower(email) = ?`, `WHERE date(created_at) = ?`, comparing a string column to a number | Rewrite as a range (`created_at >= ? AND created_at < ?`), store a normalized column, or add an expression index |
| Leading wildcard: `LIKE '%abc'` | Full-text search, trigram index (Postgres `pg_trgm`), or a reversed-column index |
| `OR` across different columns | `UNION ALL` of indexed queries, or a composite index that covers both |
| `SELECT *` | Select only needed columns; enables covering indexes and cuts I/O |
| `OFFSET` pagination on large tables | Keyset pagination: `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT n` |
| Queries inside loops (N+1) | Batch with `IN (...)`, a join, or the ORM's eager loading |
| No `LIMIT` on list queries | Add a limit and pagination |
| `COUNT(*)` on large tables for UI totals | Approximate counts, cached counters, or "has more" instead of totals |
| Transaction held open across network calls | Do the remote call before or after the transaction |
| `NOT IN (subquery)` with nullable columns | `NOT EXISTS` (correct with NULLs and usually faster) |

If the user can run queries against a non-production copy, ask for the plan: `EXPLAIN (ANALYZE, BUFFERS)` in Postgres, `EXPLAIN ANALYZE` in MySQL 8. Interpret it: sequential scans on large tables, row estimate vs actual mismatches (stale statistics), sorts spilling to disk, nested loops over large inputs. **Never run `EXPLAIN ANALYZE` on writes (`INSERT/UPDATE/DELETE`) against production**: it executes the statement.

Only report problems you can point to in the code; label speculative ones "worth measuring".

## Phase 5. Safe migrations

Check every migration that touches an existing table. A migration that's fine on an empty dev table can lock a production table for minutes.

| Operation | Safe approach |
|-----------|---------------|
| Create an index on an existing table | Postgres: `CREATE INDEX CONCURRENTLY` (must run outside a transaction; most tools need a flag for this). MySQL: `ALGORITHM=INPLACE, LOCK=NONE`, or gh-ost / pt-online-schema-change for very large tables. |
| Add a `NOT NULL` column | Postgres 11+ / MySQL 8: adding with a constant default is instant. Otherwise: add nullable → backfill in batches → add the constraint. In Postgres, add `CHECK (...) NOT VALID` then `VALIDATE CONSTRAINT` to avoid a long lock. |
| Add a foreign key | Postgres: `ADD CONSTRAINT ... NOT VALID`, then `VALIDATE CONSTRAINT` separately |
| Rename or drop a column | Expand/contract: add the new column, dual-write, backfill, switch reads, deploy, then drop the old column in a later migration. Running app versions must work with both schemas. |
| Change a column type | Usually rewrites the table. Use expand/contract with a new column. |
| Backfill large tables | Batches of a few thousand rows by primary key range, with a pause between batches, outside the schema migration |
| Any migration | Has a down/rollback step, or a note explaining why it can't be reversed |

Recipes per engine in [REFERENCE.md](REFERENCE.md#online-migration-recipes). Set a lock timeout (`SET lock_timeout = '5s'` in Postgres) so a blocked migration fails fast instead of queueing every query behind it.

## Phase 6. Output

Present suggestions as a numbered list grouped by table or file, each with a location and the reason:

```
### db-design suggestions

**orders** (migrations/20261122_create_orders.sql)
1. [table] id is BIGSERIAL; fine. Add updated_at trigger, column has no auto-update (line 9)
2. [column] status ENUM('pending','paid') → text + CHECK constraint; adding 'refunded' later would need an enum migration (line 6)
3. [column] amount FLOAT → NUMERIC(19,4); floats round currency (line 7)
4. [fk] customer_id REFERENCES customers ON DELETE CASCADE → RESTRICT, and add index on customer_id; Postgres doesn't create it and cascades would scan orders (line 8)
5. [index] (created_at, customer_id) → (customer_id, created_at DESC) per ESR; serves ListByCustomer in orders/repo.go:42
6. [migration] CREATE INDEX on existing table → CONCURRENTLY (line 14)

Do you want ascending/descending indexes on created_at / updated_at?
Apply which? (all / numbers / none)
```

Apply the approved changes in the repo's migration/ORM format, matching neighboring files (naming, file layout, up/down structure). For Design mode, output the full table definition. Then finish with:

```
## DB Design: <table(s)>

| Area       | Status            | Notes |
|------------|-------------------|-------|
| Tables     | ✅ / ⚠️ / ❌ / n/a | e.g. standard columns present, updated_at trigger added |
| Columns    | ...               | |
| Indexes    | ...               | e.g. 2 added (ESR), 1 redundant dropped |
| Queries    | ...               | |
| Migrations | ...               | e.g. index built concurrently, rollback included |

**Applied:** <changes, with files>
**Still open:** <skipped suggestions, things to verify against real data (e.g. confirm index unused in prod stats)>
```

Don't run migrations against a real database, and don't commit; leave that to the user.

## Principles

- **Repo conventions over textbook.** If the repo's pattern differs from the advice here, follow the repo and mention the tradeoff once.
- **Access patterns drive design.** Indexes and NoSQL keys exist to serve specific queries; name the query each one serves.
- **Explain the cost.** Every "avoid" (enum, FK, cascade, extra index) comes with the concrete cost, so the user can make an informed call.
- **Cite locations.** Every suggestion names a file and line, and every convention cites an existing example.
- **Be proportional.** A 50-row config table doesn't need an index audit or an online migration.
- **Ask before editing.** Suggestions are a menu; the user picks.
