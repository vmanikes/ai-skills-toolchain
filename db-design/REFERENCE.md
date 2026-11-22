# DB Design Reference

Per-engine details for the workflow in [SKILL.md](SKILL.md). Check the engine version from Phase 0 before using version-specific syntax.

## Convention discovery

Where to find the engine, migration tool and existing schema. Use `rg` if available, otherwise `grep -r`.

| Look for | Where / pattern |
|----------|-----------------|
| Engine | `docker-compose*.yml` (`image: postgres:16`), `DATABASE_URL` in `.env.example`, driver dependencies (`pg`, `psycopg`, `mysql2`, `pgx`, `go-sql-driver/mysql`, `@aws-sdk/client-dynamodb`, `mongodb`) |
| Prisma | `prisma/schema.prisma`, `prisma/migrations/` |
| Django | `*/models.py`, `*/migrations/0*.py` |
| Alembic / SQLAlchemy | `alembic.ini`, `alembic/versions/`, `Mapped[`, `Column(` |
| ActiveRecord | `db/migrate/`, `db/schema.rb` or `db/structure.sql` |
| TypeORM / Drizzle / Sequelize / Knex | `@Entity(`, `pgTable(` / `mysqlTable(`, `sequelize.define`, `knex.schema` |
| Go | `migrations/*.up.sql` (golang-migrate), `-- +goose Up` (goose), `ent/schema/`, `sqlc.yaml` |
| JVM | `db/migration/V*__*.sql` (Flyway), `changelog*.xml/yaml` (Liquibase), `@Entity` / `@Table` |
| DynamoDB | `AWS::DynamoDB::Table` (CloudFormation/SAM), `aws_dynamodb_table` (Terraform), `new Table(` (CDK), `KeySchema`, `GlobalSecondaryIndexes` |
| MongoDB | `createIndex(`, `new Schema(` (Mongoose), `@Document` |

Query patterns for **Optimize indexes** mode:

```bash
rg -n --type-add 'code:*.{go,py,rb,ts,js,java,kt,cs,php}' -t code \
  -e 'SELECT .* FROM <table>' -e 'FROM\s+<table>' -e 'JOIN\s+<table>' \
  -e '<Model>\.(objects|query|where|find|filter|findMany|findFirst|findAll)' \
  -e 'Table\(["\x27]<table>' -i
```

Also check `ORDER BY`, `order_by`, `orderBy` and pagination helpers near each match, since sort order affects index choice.

## ID types by engine

| Engine | Recommended `id` | How to generate | Notes |
|--------|------------------|-----------------|-------|
| Postgres | `uuid` (UUIDv7) or `bigint GENERATED ALWAYS AS IDENTITY` | `uuidv7()` built in from Postgres 18; earlier versions generate in the app (or the `pg_uuidv7` extension) | Postgres has no unsigned integers; `bigint` identity is the equivalent. Avoid `gen_random_uuid()` (v4) as a PK on large tables. |
| MySQL / MariaDB | `BIGINT UNSIGNED AUTO_INCREMENT` or `BINARY(16)` UUIDv7 / ULID | App-generated UUIDv7/ULID stored with `UUID_TO_BIN()` | InnoDB clusters rows by PK, so random UUIDs hurt more here than anywhere: every insert lands on a random page. Never store UUIDs as `CHAR(36)`. |
| SQLite | `INTEGER PRIMARY KEY` | Automatic (rowid alias) | Add `AUTOINCREMENT` only if ids must never be reused. |
| SQL Server | `BIGINT IDENTITY` or `uniqueidentifier DEFAULT NEWSEQUENTIALID()` | Built in | `NEWID()` is random; avoid as a clustered key. |
| DynamoDB | String partition key, e.g. `USER#<ulid>` | App-generated ULID / UUID | Partition key is hashed, so ULIDs spread fine. Watch low-cardinality GSI keys instead. |
| MongoDB | `ObjectId` | Driver default | Time-ordered. If sharding on `_id`, use a hashed shard key. |
| Cassandra / ScyllaDB | `uuid` or `timeuuid` | `uuid()` / `now()` | Partition key is hashed (Murmur3). Use `timeuuid` as a clustering column for time ordering. |
| Spanner | `STRING(36)` UUIDv4, or bit-reversed sequence | `GENERATE_UUID()`, `GET_NEXT_SEQUENCE_VALUE` on a `bit_reversed_positive` sequence | Range-partitioned: monotonic keys create a write hotspot. |
| CockroachDB | `UUID DEFAULT gen_random_uuid()` | Built in | Range-partitioned: random UUIDs spread writes; avoid `SERIAL` as the leading key. Hash-sharded indexes help for sequential secondary keys. |
| Bigtable / HBase | Row key with a non-monotonic prefix | Hash or field-promoted prefix | Never start the row key with a timestamp. |

## `updated_at` mechanisms

**Postgres** (one shared function, one trigger per table):

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_set_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

**MySQL:**

```sql
updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
```

**SQL Server / SQLite:** an `AFTER UPDATE` trigger, or the ORM.

**ORMs:** Prisma `@updatedAt`, Django `auto_now=True`, SQLAlchemy `onupdate=func.now()`, ActiveRecord `t.timestamps` (automatic), TypeORM `@UpdateDateColumn()`, Sequelize `timestamps: true`, Ent `field.Time("updated_at").UpdateDefault(time.Now)`.

ORM-managed timestamps don't fire for raw SQL updates or bulk updates (`QuerySet.update()`, `updateMany`). Mention this when the codebase does bulk updates; a DB trigger covers every path.

## Enum alternatives

**Lookup table** (values can be added, renamed or retired without a schema change):

```sql
CREATE TABLE order_statuses (
  code        text PRIMARY KEY,          -- 'pending', 'paid', 'shipped'
  label       text NOT NULL,
  sort_order  int  NOT NULL DEFAULT 0,
  is_active   boolean NOT NULL DEFAULT true
);

ALTER TABLE orders ADD COLUMN status text NOT NULL REFERENCES order_statuses(code);
```

A text code as the lookup key keeps rows readable without a join. If FKs are avoided on a high-write table, keep the lookup table and validate in the app.

**CHECK constraint:**

```sql
-- Postgres, SQLite, SQL Server, MySQL 8.0.16+
status text NOT NULL CONSTRAINT orders_status_check
  CHECK (status IN ('pending', 'paid', 'shipped'))

-- Changing the allowed values
ALTER TABLE orders DROP CONSTRAINT orders_status_check;
ALTER TABLE orders ADD CONSTRAINT orders_status_check
  CHECK (status IN ('pending', 'paid', 'shipped', 'refunded')) NOT VALID;  -- Postgres: skip the full scan
ALTER TABLE orders VALIDATE CONSTRAINT orders_status_check;
```

MySQL before 8.0.16 parses `CHECK` but ignores it; use a lookup table there.

**Why native enums hurt:**
- Postgres: `ALTER TYPE ... ADD VALUE` works, but values can't be removed or reordered without recreating the type and rewriting every column that uses it.
- MySQL: changing an `ENUM` definition (except appending at the end) rebuilds the table. Sorting is by declaration index, not alphabetically.
- Every new value is a deploy-coordinated schema migration.

## ESR examples

**Feed with filter and sort:**

```sql
-- Query
SELECT id, title FROM posts
WHERE author_id = $1 AND published_at > $2
ORDER BY published_at DESC LIMIT 20;

-- ✅ (author_id, published_at DESC): E then S; the range is on the sort column, so one index range read, already sorted
-- ❌ (published_at, author_id): leading range; scans every author's posts in the window
```

**Equality + sort + range on different columns:**

```sql
-- Query
SELECT * FROM tickets
WHERE project_id = $1 AND status = 'open' AND priority >= 3
ORDER BY created_at DESC LIMIT 50;

-- ✅ (project_id, status, created_at DESC, priority): E, E, S, R. Rows come out sorted; priority filtered from the index
-- ⚠️ (project_id, status, priority, created_at): E, E, R, S. Narrower scan, but needs a sort. Can win if the range is very selective; check EXPLAIN.
```

ESR is the default rule, not a law. When a range filter is extremely selective and the result is small, E-R-S can beat E-S-R. Say so when relevant and suggest comparing plans.

**MongoDB** (where ESR originates):

```js
// db.orders.find({ customerId: X, total: { $gt: 100 } }).sort({ createdAt: -1 })
db.orders.createIndex({ customerId: 1, createdAt: -1, total: 1 })   // ✅ E, S, R
```

**IN lists:** a small `IN (...)` acts like equality; a large one behaves more like a range for sorting purposes.

## Partial index support

| Engine | Syntax | Notes |
|--------|--------|-------|
| Postgres | `CREATE INDEX idx_orders_shipped_at ON orders (shipped_at) WHERE shipped_at IS NOT NULL;` | Query `WHERE` must imply the predicate for the planner to use it |
| SQLite | Same as Postgres | 3.8.0+ |
| SQL Server | `CREATE INDEX ... ON orders (shipped_at) WHERE shipped_at IS NOT NULL;` (filtered index) | Parameterized queries may not match the filter; check plans |
| MongoDB | `createIndex({ shippedAt: 1 }, { partialFilterExpression: { shippedAt: { $exists: true } } })` | Or a sparse index |
| MySQL / MariaDB | Not supported | Options: skip, or a functional index on a generated column. InnoDB does store NULLs in secondary indexes. |
| DynamoDB | Sparse GSI | A GSI only contains items that have the GSI key attribute, so leaving it unset excludes the item |

Common patterns:

```sql
-- Soft delete: index and enforce uniqueness only on live rows
CREATE UNIQUE INDEX uq_users_email_live ON users (tenant_id, email) WHERE deleted_at IS NULL;

-- Work queue: only the small set of unprocessed rows
CREATE INDEX idx_jobs_pending ON jobs (run_at) WHERE status = 'pending';
```

## Index audit queries

**Postgres: unused indexes** (stats since last reset; check every replica, as each tracks its own):

```sql
SELECT s.schemaname, s.relname AS table, s.indexrelname AS index,
       s.idx_scan, pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0
  AND NOT i.indisunique        -- unique indexes enforce constraints even if never scanned
  AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;

SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
```

**Postgres: duplicate and prefix-redundant indexes:**

```sql
SELECT a.indrelid::regclass AS table,
       a.indexrelid::regclass AS redundant_index,
       b.indexrelid::regclass AS covered_by
FROM pg_index a
JOIN pg_index b
  ON a.indrelid = b.indrelid
 AND a.indexrelid <> b.indexrelid
 AND array_to_string(b.indkey, ' ') LIKE array_to_string(a.indkey, ' ') || '%'
 AND (a.indkey <> b.indkey OR a.indexrelid > b.indexrelid)   -- report one side of exact duplicates
WHERE NOT a.indisunique
  AND a.indpred IS NULL AND b.indpred IS NULL                  -- skip partial indexes
  AND a.indexprs IS NULL AND b.indexprs IS NULL;               -- skip expression indexes
```

Treat results as candidates: column sort order, operator class and index type (B-tree vs GIN) also matter.

**MySQL 8 (sys schema):**

```sql
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = DATABASE();
SELECT * FROM sys.schema_redundant_indexes WHERE table_schema = DATABASE();
```

**SQL Server:** `sys.dm_db_index_usage_stats` (reset on restart) and `sys.dm_db_missing_index_details` (suggestions only, often over-eager).

**MongoDB:** `db.collection.aggregate([{ $indexStats: {} }])`.

## Query anti-patterns

| Instead of | Write | Why |
|------------|-------|-----|
| `WHERE date(created_at) = '2026-11-22'` | `WHERE created_at >= '2026-11-22' AND created_at < '2026-11-23'` | Function on column prevents index use |
| `WHERE lower(email) = $1` | Store a normalized column, use `citext` (Postgres), a case-insensitive collation (MySQL default), or `CREATE INDEX ON users (lower(email))` | Same |
| `WHERE phone = 5551234` on a `varchar` column | `WHERE phone = '5551234'` | Implicit cast on the column side disables the index (MySQL) |
| `WHERE name LIKE '%smith'` | Trigram index (`pg_trgm` + GIN), full-text search, or a reversed column | Leading wildcard can't use a B-tree |
| `WHERE a = 1 OR b = 2` | `SELECT ... WHERE a = 1 UNION ALL SELECT ... WHERE b = 2 AND a IS DISTINCT FROM 1` | Each branch can use its own index (Postgres can also do bitmap OR) |
| `ORDER BY created_at LIMIT 20 OFFSET 100000` | `WHERE (created_at, id) < ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 20` | OFFSET reads and discards every skipped row |
| `WHERE id NOT IN (SELECT user_id FROM bans)` | `WHERE NOT EXISTS (SELECT 1 FROM bans b WHERE b.user_id = u.id)` | `NOT IN` returns nothing if the subquery has a NULL |
| `SELECT COUNT(*) FROM events` for a UI badge | `reltuples` estimate (Postgres), a counter table, or `LIMIT n+1` to show "n+" | Exact counts scan the table (or an index) |
| Loop of `SELECT ... WHERE id = ?` | `WHERE id = ANY($1)` / `IN (...)`, or eager loading (`select_related`, `includes`, `include`, `Preload`) | N+1 round trips |
| `SELECT *` in hot paths | Explicit column list | Enables index-only scans; less I/O and network |

**Reading `EXPLAIN` (Postgres):**
- `Seq Scan` on a large table with a selective filter → missing or unusable index.
- `rows=` estimate far from `actual rows` → stale statistics; suggest `ANALYZE <table>`.
- `Sort Method: external merge  Disk:` → sort spilled to disk; an index matching `ORDER BY` avoids the sort.
- `Rows Removed by Filter` high on an `Index Scan` → index columns in the wrong order (check ESR) or missing a column.
- `Buffers: shared read` high → data not cached; covering index or fewer columns helps.

**MySQL:** in `EXPLAIN`, `type: ALL` is a full scan; `Extra: Using filesort` / `Using temporary` mean extra sort/temp-table work; `key: NULL` means no index used.

## Online migration recipes

**Postgres:**

```sql
SET lock_timeout = '5s';           -- fail fast instead of blocking all queries behind the lock
SET statement_timeout = '15min';

-- Index on an existing table (not inside a transaction block)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_created
  ON orders (customer_id, created_at DESC);
-- If it fails, it leaves an INVALID index: DROP INDEX CONCURRENTLY and retry.

-- NOT NULL on an existing column without a long lock
ALTER TABLE orders ADD CONSTRAINT orders_region_not_null CHECK (region IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT orders_region_not_null;
ALTER TABLE orders ALTER COLUMN region SET NOT NULL;   -- PG12+ uses the validated CHECK, no scan
ALTER TABLE orders DROP CONSTRAINT orders_region_not_null;

-- Foreign key without a long lock
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer
  FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;
```

Tool flags for running outside a transaction: Rails `disable_ddl_transaction!` + `algorithm: :concurrently`; Django `AddIndexConcurrently` with `atomic = False`; Alembic `with op.get_context().autocommit_block():` + `postgresql_concurrently=True`; golang-migrate: put the statement alone in its own migration file; Flyway: `executeInTransaction=false` in the script's config; Prisma: edit the generated migration SQL to add `CONCURRENTLY`, alone in its own migration.

**MySQL 8:**

```sql
ALTER TABLE orders ADD COLUMN region VARCHAR(32) NULL, ALGORITHM=INSTANT;
ALTER TABLE orders ADD INDEX idx_orders_customer_created (customer_id, created_at),
  ALGORITHM=INPLACE, LOCK=NONE;
```

Specifying `ALGORITHM`/`LOCK` makes MySQL error out instead of silently falling back to a blocking copy. For very large or replicated tables, use `gh-ost` or `pt-online-schema-change`.

**Batched backfill:**

```sql
-- Repeat until 0 rows updated; pause briefly between batches to let replicas catch up
UPDATE orders SET region = 'us'
WHERE id IN (
  SELECT id FROM orders WHERE region IS NULL ORDER BY id LIMIT 5000
);
```

Run backfills as a separate script or job, not inside the schema migration, so a slow backfill doesn't hold migration locks.

**Expand/contract (rename `name` → `full_name`):**

1. Migration: add `full_name` nullable.
2. Deploy: write both columns, read `name`.
3. Backfill `full_name` from `name` in batches.
4. Deploy: read `full_name`.
5. Deploy: stop writing `name`.
6. Migration: drop `name` (and add `NOT NULL` to `full_name` using the recipe above).
