# Database Migrations

## Table of Contents

1. [Overview](#overview)
   - [What Are Schema Migrations?](#what-are-schema-migrations)
   - [Why Migrations Matter](#why-migrations-matter)
   - [The Migration Lifecycle](#the-migration-lifecycle)
2. [Migration Tools](#migration-tools)
   - [Tool Comparison](#tool-comparison)
   - [Flyway](#flyway)
   - [Liquibase](#liquibase)
   - [golang-migrate](#golang-migrate)
   - [Alembic (Python)](#alembic-python)
   - [Entity Framework Migrations (.NET)](#entity-framework-migrations-net)
   - [Rails ActiveRecord Migrations](#rails-activerecord-migrations)
   - [Prisma Migrate](#prisma-migrate)
3. [Migration Best Practices](#migration-best-practices)
   - [Idempotent Migrations](#idempotent-migrations)
   - [Version Numbering](#version-numbering)
   - [One Change per Migration](#one-change-per-migration)
   - [Review Process](#review-process)
4. [Zero-Downtime Migrations](#zero-downtime-migrations)
   - [The Expand-Contract Pattern](#the-expand-contract-pattern)
   - [Adding a Column (Safe)](#adding-a-column-safe)
   - [Renaming a Column (Expand-Contract)](#renaming-a-column-expand-contract)
   - [Changing a Column Type (Expand-Contract)](#changing-a-column-type-expand-contract)
   - [Dropping a Column (Deploy Code First)](#dropping-a-column-deploy-code-first)
   - [Adding a NOT NULL Constraint (Backfill First)](#adding-a-not-null-constraint-backfill-first)
5. [Large Table Migrations](#large-table-migrations)
   - [pt-online-schema-change (MySQL)](#pt-online-schema-change-mysql)
   - [pg_repack (PostgreSQL)](#pg_repack-postgresql)
   - [Online DDL](#online-ddl)
   - [Batched Backfills](#batched-backfills)
6. [Data Migrations vs Schema Migrations](#data-migrations-vs-schema-migrations)
   - [When to Separate Them](#when-to-separate-them)
   - [Backfill Strategies](#backfill-strategies)
7. [Rollback Strategies](#rollback-strategies)
   - [Forward-Only vs Reversible Migrations](#forward-only-vs-reversible-migrations)
   - [Blue-Green Database Deployments](#blue-green-database-deployments)
8. [Schema Evolution and Compatibility (DDIA Ch.4)](#schema-evolution-and-compatibility-ddia-ch4)
   - [Forward and Backward Compatibility](#forward-and-backward-compatibility)
   - [Encoding Formats and Schema Registries](#encoding-formats-and-schema-registries)
9. [Version History](#version-history)

---

## Overview

### What Are Schema Migrations?

A **schema migration** is a versioned, incremental change to a database's structure — tables, columns, indexes, constraints, or stored procedures. Migrations transform a database from one known state to the next, forming a linear history of every structural change since the database was created.

Think of migrations as **version control for your database schema**. Just as Git tracks changes to source code, migration files track changes to the database. Each migration is a small, atomic unit of change with an up (apply) and optionally a down (revert) operation.

```
  Migration History
  ─────────────────

  V1              V2              V3              V4
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Create   │───▶│ Add      │───▶│ Create   │───▶│ Add      │
  │ users    │    │ email to │    │ orders   │    │ index on │
  │ table    │    │ users    │    │ table    │    │ orders   │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### Why Migrations Matter

| Problem Without Migrations | How Migrations Solve It |
|---|---|
| Schema changes applied ad-hoc via manual SQL | Every change is versioned, reviewed, and repeatable |
| No record of what changed or when | Full audit trail in source control |
| Dev, staging, and production schemas drift apart | Same migrations run in every environment |
| Rollbacks require guessing what to undo | Down migrations or compensating changes are pre-planned |
| Multiple developers step on each other's changes | Migration ordering and conflict detection built into tooling |
| Impossible to recreate a database from scratch | Run all migrations sequentially to build any environment |

### The Migration Lifecycle

```
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │  1. Author   │────▶│  2. Review   │────▶│  3. Test      │────▶│  4. Apply    │
  │              │     │              │     │              │     │              │
  │  Write the   │     │  Code review │     │  Run against │     │  Run in      │
  │  migration   │     │  + DBA sign  │     │  staging or  │     │  production  │
  │  file        │     │  off         │     │  CI database │     │              │
  └──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
         │                                                              │
         │                    ┌──────────────┐                          │
         │                    │  5. Record   │◀─────────────────────────┘
         │                    │              │
         │                    │  Mark as     │
         │                    │  applied in  │
         │                    │  tracking    │
         │                    │  table       │
         │                    └──────────────┘
         │                           │
         ▼                           ▼
  ┌────────────────────────────────────────┐
  │         Migration Tracking Table       │
  │                                        │
  │  version  │ applied_at       │ success │
  │  ─────────┼──────────────────┼──────── │
  │  V001     │ 2025-01-15 10:30 │ true    │
  │  V002     │ 2025-01-22 14:00 │ true    │
  │  V003     │ 2025-02-01 09:15 │ true    │
  └────────────────────────────────────────┘
```

Every migration tool maintains a **tracking table** (often called `schema_migrations`, `flyway_schema_history`, or `__EFMigrationsHistory`) that records which migrations have been applied. This ensures each migration runs exactly once and in order.

---

## Migration Tools

### Tool Comparison

| Tool | Language / Ecosystem | Migration Format | Approach | Rollback Support | Key Strength |
|---|---|---|---|---|---|
| **Flyway** | Java / JVM | SQL files | Version-based | Paid (Teams) | Simple, SQL-first |
| **Liquibase** | Java / JVM | XML, YAML, JSON, SQL | Changelog-based | Built-in | Database-agnostic changelogs |
| **golang-migrate** | Go | SQL files | Version-based | Built-in (down files) | Lightweight, CLI-first |
| **Alembic** | Python / SQLAlchemy | Python scripts | Version-based | Built-in (`downgrade`) | Auto-generate from models |
| **EF Migrations** | .NET / C# | C# code | Code-first | Built-in (`Down` method) | Tight ORM integration |
| **ActiveRecord** | Ruby / Rails | Ruby DSL | Timestamp-based | Built-in (`down`/`change`) | Convention over configuration |
| **Prisma Migrate** | TypeScript / Node.js | SQL files | Declarative schema | Limited (no auto down) | Schema-driven, type-safe |

### Flyway

Flyway uses **numbered SQL files** with a strict naming convention. Migrations are immutable once applied — you cannot edit an applied migration.

```
db/migration/
├── V1__create_users_table.sql
├── V2__add_email_to_users.sql
├── V3__create_orders_table.sql
└── V4__add_index_on_orders_user_id.sql
```

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    username    VARCHAR(255) NOT NULL UNIQUE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- V2__add_email_to_users.sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
CREATE UNIQUE INDEX idx_users_email ON users (email);
```

```bash
# Apply all pending migrations
flyway migrate

# Check current version
flyway info

# Validate applied migrations match local files
flyway validate
```

### Liquibase

Liquibase uses a **changelog** model. Changes are defined as changesets, each with a unique `id` and `author`. It supports multiple formats but YAML and SQL are most common.

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: alice
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: username
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true

  - changeSet:
      id: 2
      author: alice
      changes:
        - addColumn:
            tableName: users
            columns:
              - column:
                  name: email
                  type: VARCHAR(255)
      rollback:
        - dropColumn:
            tableName: users
            columnName: email
```

### golang-migrate

A lightweight, CLI-driven tool popular in Go projects. Each migration is a pair of `up` and `down` SQL files.

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_add_email.up.sql
└── 000002_add_email.down.sql
```

```bash
# Apply all pending migrations
migrate -path ./migrations -database "postgres://localhost/mydb?sslmode=disable" up

# Roll back the last migration
migrate -path ./migrations -database "postgres://localhost/mydb?sslmode=disable" down 1

# Go to a specific version
migrate -path ./migrations -database "postgres://localhost/mydb?sslmode=disable" goto 3
```

### Alembic (Python)

Alembic integrates with SQLAlchemy and can **auto-generate** migrations by comparing model definitions against the current database state.

```bash
# Initialize Alembic
alembic init migrations

# Auto-generate a migration from model changes
alembic revision --autogenerate -m "add email to users"

# Apply all pending migrations
alembic upgrade head

# Roll back one migration
alembic downgrade -1
```

```python
# migrations/versions/a1b2c3d4_add_email_to_users.py
"""add email to users"""
revision = 'a1b2c3d4'
down_revision = '00000001'

from alembic import op
import sqlalchemy as sa

def upgrade():
    op.add_column('users', sa.Column('email', sa.String(255)))
    op.create_unique_constraint('uq_users_email', 'users', ['email'])

def downgrade():
    op.drop_constraint('uq_users_email', 'users')
    op.drop_column('users', 'email')
```

### Entity Framework Migrations (.NET)

EF Core migrations are generated from C# model classes. The tooling diffs your models against a snapshot and produces migration code.

```bash
# Generate a migration from model changes
dotnet ef migrations add AddEmailToUsers

# Apply to database
dotnet ef database update

# Roll back to a specific migration
dotnet ef database update CreateUsersTable

# Generate a SQL script instead of applying directly
dotnet ef migrations script --idempotent -o migration.sql
```

```csharp
// Migrations/20250115_AddEmailToUsers.cs
public partial class AddEmailToUsers : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "Email",
            table: "Users",
            type: "varchar(255)",
            nullable: true);

        migrationBuilder.CreateIndex(
            name: "IX_Users_Email",
            table: "Users",
            column: "Email",
            unique: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(name: "IX_Users_Email", table: "Users");
        migrationBuilder.DropColumn(name: "Email", table: "Users");
    }
}
```

### Rails ActiveRecord Migrations

Rails migrations use a Ruby DSL with timestamp-based ordering. The `change` method supports automatic reversibility for common operations.

```bash
# Generate a migration
rails generate migration AddEmailToUsers email:string:uniq

# Apply all pending migrations
rails db:migrate

# Roll back the last migration
rails db:rollback

# Roll back the last 3 migrations
rails db:rollback STEP=3
```

```ruby
# db/migrate/20250115103000_add_email_to_users.rb
class AddEmailToUsers < ActiveRecord::Migration[7.1]
  def change
    add_column :users, :email, :string, limit: 255
    add_index :users, :email, unique: true
  end
end
```

### Prisma Migrate

Prisma takes a **declarative** approach — you edit a schema file and the tool generates SQL migrations from the diff.

```prisma
// prisma/schema.prisma
model User {
  id        Int      @id @default(autoincrement())
  username  String   @unique @db.VarChar(255)
  email     String?  @unique @db.VarChar(255)   // ← added
  createdAt DateTime @default(now())
}
```

```bash
# Generate and apply a migration
npx prisma migrate dev --name add_email_to_users

# Apply pending migrations in production (no prompts)
npx prisma migrate deploy

# Reset the database (destructive — dev only)
npx prisma migrate reset
```

---

## Migration Best Practices

### Idempotent Migrations

An **idempotent migration** can be applied multiple times without error or side effects. This is critical for retry logic and environments where the tracking table might be out of sync.

```sql
-- ✅ Idempotent — safe to re-run
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL
);

ALTER TABLE users ADD COLUMN IF NOT EXISTS email VARCHAR(255);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users (email);
```

```sql
-- ❌ Not idempotent — fails on second run
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL
);
-- ERROR: relation "users" already exists
```

> **Tip:** PostgreSQL supports `IF NOT EXISTS` / `IF EXISTS` on most DDL. MySQL supports it on `CREATE TABLE` and `DROP TABLE` but has weaker support on `ALTER TABLE`. Write wrapper scripts or use tool-specific guards for MySQL.

### Version Numbering

| Strategy | Format | Pros | Cons |
|---|---|---|---|
| **Sequential** | `V001`, `V002`, `V003` | Clear ordering, easy to scan | Merge conflicts when two developers pick the same number |
| **Timestamp** | `20250115103000` | No conflicts across branches | Harder to scan visually |
| **Hybrid** | `V001_20250115` | Clear ordering + timestamp context | Slightly verbose |

Most teams use **timestamp-based** numbering in high-velocity projects to avoid conflicts. For smaller teams, sequential numbering is simpler.

### One Change per Migration

Each migration should perform **exactly one logical change**. This makes migrations easier to review, debug, and roll back.

❌ **Don't:**

- Create a table, add indexes, insert seed data, and alter another table in one migration
- Mix schema changes with data changes

✅ **Do:**

- One migration to create the table
- One migration to add the index
- A separate data migration for backfills

### Review Process

Treat migration files with the same rigor as production code:

1. **Code review** — every migration goes through pull request review
2. **DBA review** — for migrations touching large tables (>1M rows) or adding constraints
3. **CI validation** — run migrations against a test database in CI
4. **Dry run** — generate the SQL without executing to review the plan
5. **Staging first** — always apply to a staging environment before production
6. **Lock awareness** — annotate migrations that acquire table locks

```sql
-- Migration: V042__add_status_to_orders.sql
-- Reviewed by: @dba-team
-- Lock level: ACCESS EXCLUSIVE (brief — column add only)
-- Estimated time: < 1s on 5M row table
-- Rollback: V042_rollback__remove_status_from_orders.sql

ALTER TABLE orders ADD COLUMN status VARCHAR(50) DEFAULT 'pending';
```

---

## Zero-Downtime Migrations

In production systems with continuous deployments, database migrations must not cause downtime. The core challenge: **application code and database schema must remain compatible throughout the deployment window**, when old and new versions of the application may run simultaneously.

> **Key insight (DDIA Ch.4):** Rolling deployments mean your system must handle both the old and new schema at the same time. This is the same forward/backward compatibility problem that Kleppmann describes for data encoding formats — and the solution is similar: evolve the schema in stages.

### The Expand-Contract Pattern

The **expand-contract pattern** (also called **parallel change**) is the primary technique for zero-downtime schema changes. It splits a breaking migration into multiple non-breaking steps:

```
  Expand-Contract Pattern
  ───────────────────────

  Phase 1: EXPAND                Phase 2: MIGRATE              Phase 3: CONTRACT
  ┌──────────────────────┐       ┌──────────────────────┐       ┌──────────────────────┐
  │ Add new structure    │──────▶│ Deploy code that     │──────▶│ Remove old structure │
  │ alongside old one    │       │ uses new structure   │       │ once nothing reads   │
  │                      │       │ + backfill data      │       │ the old one          │
  │ Old code still works │       │                      │       │                      │
  │ New code still works │       │ Old code still works │       │ Only new code runs   │
  └──────────────────────┘       └──────────────────────┘       └──────────────────────┘
```

### Adding a Column (Safe)

Adding a nullable column with no default is **safe in most databases** — it acquires a brief metadata lock and does not rewrite the table.

```sql
-- Step 1: Add the column (safe, no rewrite)
ALTER TABLE users ADD COLUMN phone VARCHAR(50);

-- Step 2: Deploy code that writes to the new column
-- Step 3: Backfill existing rows (see "Batched Backfills")
-- Step 4: Optionally add a NOT NULL constraint later
```

> **PostgreSQL note:** Since PostgreSQL 11, `ALTER TABLE ... ADD COLUMN ... DEFAULT <value>` no longer rewrites the table. The default is stored in the catalog. This makes adding columns with defaults safe in PostgreSQL but still dangerous in older MySQL versions.

### Renaming a Column (Expand-Contract)

Renaming a column directly (`ALTER TABLE ... RENAME COLUMN`) breaks any running application code that references the old name. Use expand-contract instead:

```
  Timeline
  ────────────────────────────────────────────────────────────────────

  Deploy 1: EXPAND                 Deploy 2: SWITCH           Deploy 3: CONTRACT
  ┌─────────────────────┐          ┌────────────────────┐     ┌────────────────────┐
  │ Add new column      │          │ Code reads/writes  │     │ Drop old column    │
  │ Copy data via       │          │ new column only    │     │ Drop trigger       │
  │ trigger or backfill │          │                    │     │                    │
  │ Code writes to both │          │ Trigger still      │     │                    │
  └─────────────────────┘          │ syncs for safety   │     └────────────────────┘
                                   └────────────────────┘
```

**Step-by-step:**

```sql
-- Deploy 1: EXPAND — add new column and sync trigger
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

UPDATE users SET full_name = name WHERE full_name IS NULL;

CREATE OR REPLACE FUNCTION sync_name_to_full_name()
RETURNS TRIGGER AS $$
BEGIN
    NEW.full_name := NEW.name;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_name
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_name_to_full_name();
```

```sql
-- Deploy 2: SWITCH — application now uses full_name exclusively
-- (code change only — no schema change in this step)
```

```sql
-- Deploy 3: CONTRACT — remove old column and trigger
DROP TRIGGER trg_sync_name ON users;
DROP FUNCTION sync_name_to_full_name();
ALTER TABLE users DROP COLUMN name;
```

### Changing a Column Type (Expand-Contract)

Changing a column type (e.g., `INTEGER` → `BIGINT`, or `VARCHAR(50)` → `TEXT`) often requires a full table rewrite and exclusive lock. Use expand-contract:

```sql
-- Deploy 1: EXPAND — add new column with target type
ALTER TABLE orders ADD COLUMN amount_v2 NUMERIC(20, 4);

-- Backfill in batches (see "Batched Backfills" section)
UPDATE orders SET amount_v2 = amount::NUMERIC(20, 4)
WHERE id BETWEEN 1 AND 10000;

-- Trigger to keep columns in sync during transition
CREATE OR REPLACE FUNCTION sync_amount()
RETURNS TRIGGER AS $$
BEGIN
    NEW.amount_v2 := NEW.amount::NUMERIC(20, 4);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_amount
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION sync_amount();
```

```sql
-- Deploy 2: SWITCH — code reads/writes amount_v2
```

```sql
-- Deploy 3: CONTRACT — drop old column, rename new column
DROP TRIGGER trg_sync_amount ON orders;
DROP FUNCTION sync_amount();
ALTER TABLE orders DROP COLUMN amount;
ALTER TABLE orders RENAME COLUMN amount_v2 TO amount;
```

### Dropping a Column (Deploy Code First)

Never drop a column before all application instances have stopped reading it. The sequence is **code first, schema second**.

```
  Timeline
  ──────────────────────────────────────────────────────

  Deploy 1                        Deploy 2 (days later)
  ┌───────────────────────┐       ┌───────────────────────┐
  │ Remove all references │       │ DROP COLUMN            │
  │ to the column in code │       │                        │
  │                       │       │ Safe — nothing reads   │
  │ Column still exists   │       │ or writes the column   │
  │ but is unused         │       │                        │
  └───────────────────────┘       └───────────────────────┘
```

```sql
-- Deploy 1: Code change only — remove column from SELECT, INSERT, UPDATE
-- Ensure ORM doesn't select the column (e.g., exclude from model)

-- Deploy 2: Schema change (after code deploy is fully rolled out)
ALTER TABLE users DROP COLUMN legacy_status;
```

> **Tip:** Wait at least one full deployment cycle (often 1–2 weeks) before dropping a column. This provides a safety window for rollback.

### Adding a NOT NULL Constraint (Backfill First)

Adding `NOT NULL` to an existing column fails if any rows have `NULL` values. Even if all rows are populated, some databases scan the entire table to verify — which can lock the table.

**Safe approach in PostgreSQL:**

```sql
-- Step 1: Backfill NULLs in batches
UPDATE users SET status = 'active' WHERE status IS NULL AND id BETWEEN 1 AND 10000;
UPDATE users SET status = 'active' WHERE status IS NULL AND id BETWEEN 10001 AND 20000;
-- ... continue until all NULLs are filled

-- Step 2: Add a CHECK constraint as NOT VALID (no full scan)
ALTER TABLE users ADD CONSTRAINT chk_status_not_null
    CHECK (status IS NOT NULL) NOT VALID;

-- Step 3: Validate the constraint (scans but does not hold exclusive lock in PG 12+)
ALTER TABLE users VALIDATE CONSTRAINT chk_status_not_null;

-- Step 4 (optional): Convert to a proper NOT NULL constraint
-- In PostgreSQL 12+, the planner recognizes validated CHECK constraints,
-- so this step is optional and cosmetic.
ALTER TABLE users ALTER COLUMN status SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT chk_status_not_null;
```

---

## Large Table Migrations

When tables have millions or billions of rows, even simple schema changes can take hours and lock tables for the duration. Specialized tools rewrite tables in the background without blocking reads or writes.

### pt-online-schema-change (MySQL)

Percona's `pt-online-schema-change` creates a shadow copy of the table, applies the schema change to the copy, then uses triggers to sync writes during the copy process.

```
  pt-online-schema-change Flow
  ────────────────────────────

  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
  │ 1. Create    │────▶│ 2. Copy rows in  │────▶│ 3. Swap tables   │
  │ shadow table │     │ chunks + install │     │ (atomic rename)  │
  │ with new     │     │ triggers to sync │     │                  │
  │ schema       │     │ ongoing writes   │     │ Drop old table   │
  └──────────────┘     └──────────────────┘     └──────────────────┘
```

```bash
pt-online-schema-change \
    --alter "ADD COLUMN phone VARCHAR(50)" \
    --execute \
    --chunk-size=1000 \
    --max-lag=1s \
    --check-replication-filters \
    D=mydb,t=users
```

**Key flags:**
- `--chunk-size` — number of rows copied per batch (tune for throughput vs replication lag)
- `--max-lag` — pause if replica lag exceeds this threshold
- `--critical-load` — abort if server load exceeds this threshold

### pg_repack (PostgreSQL)

`pg_repack` repacks a table online by creating a new copy in the background, then swapping the files. It does not require a trigger-based approach because it uses PostgreSQL's internal row-level logging.

```bash
# Repack a specific table (reclaim bloat + rebuild indexes)
pg_repack --table users --jobs 4 mydb

# Repack and apply a schema change (requires pg_repack 1.5+)
# For schema changes, prefer using CREATE INDEX CONCURRENTLY
# or the expand-contract pattern instead.
```

### Online DDL

Modern databases have built-in online DDL capabilities that avoid full table locks:

| Operation | PostgreSQL | MySQL 8.0+ |
|---|---|---|
| Add nullable column | ✅ Instant (metadata only) | ✅ Instant |
| Add column with default | ✅ Instant (PG 11+) | ⚠️ Copy or in-place |
| Drop column | ✅ Instant (metadata only) | ⚠️ Rebuild for some engines |
| Add index | ✅ `CREATE INDEX CONCURRENTLY` | ✅ `ALGORITHM=INPLACE` |
| Change column type | ❌ Rewrites table | ❌ Rewrites table |
| Add `NOT NULL` | ⚠️ Scans table (see workaround above) | ⚠️ Copy |

```sql
-- PostgreSQL: non-blocking index creation
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
-- This takes longer than a regular CREATE INDEX but does not lock writes.
-- IMPORTANT: Cannot be run inside a transaction block.

-- MySQL: online index creation
ALTER TABLE orders ADD INDEX idx_orders_status (status), ALGORITHM=INPLACE, LOCK=NONE;
```

### Batched Backfills

When updating millions of rows, a single `UPDATE` statement acquires a massive lock, bloats the WAL/binlog, and can cause replication lag. Batch the updates:

```sql
-- PostgreSQL: batched backfill with advisory lock
DO $$
DECLARE
    batch_size INT := 5000;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE users
        SET status = 'active'
        WHERE id IN (
            SELECT id FROM users
            WHERE status IS NULL
            ORDER BY id
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        );

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        RAISE NOTICE 'Updated % rows', rows_updated;
        PERFORM pg_sleep(0.1);  -- throttle to reduce replication lag
        COMMIT;
    END LOOP;
END $$;
```

```python
# Python batched backfill example
import time

BATCH_SIZE = 5000
SLEEP_BETWEEN_BATCHES = 0.1  # seconds

while True:
    result = db.execute("""
        UPDATE users SET status = 'active'
        WHERE id IN (
            SELECT id FROM users
            WHERE status IS NULL
            ORDER BY id
            LIMIT %s
        )
        RETURNING id
    """, [BATCH_SIZE])

    updated = result.rowcount
    db.commit()

    if updated == 0:
        break

    print(f"Updated {updated} rows")
    time.sleep(SLEEP_BETWEEN_BATCHES)
```

---

## Data Migrations vs Schema Migrations

### When to Separate Them

**Schema migrations** change the database structure (DDL): `CREATE TABLE`, `ALTER TABLE`, `CREATE INDEX`. **Data migrations** change the data itself (DML): `UPDATE`, `INSERT`, `DELETE`.

| Characteristic | Schema Migration | Data Migration |
|---|---|---|
| **SQL type** | DDL | DML |
| **Lock behavior** | Brief metadata locks (usually) | Row-level locks, potentially long |
| **Reversibility** | Often reversible (`DROP COLUMN`) | Often destructive (old values lost) |
| **Speed** | Usually fast (metadata only) | Depends on data volume |
| **When to run** | During deployment | Often decoupled from deployment |
| **Failure mode** | Transactional (DDL in PG) | May partially complete |

> **Rule of thumb:** Keep schema migrations and data migrations in **separate files**. Run schema migrations during deployment. Run data migrations as background jobs with progress tracking and retry logic.

### Backfill Strategies

```
  Backfill Decision Tree
  ──────────────────────

  How many rows need updating?
        │
        ├── < 10,000 rows
        │       └── Inline UPDATE in migration (fast, simple)
        │
        ├── 10,000 – 1,000,000 rows
        │       └── Batched UPDATE in migration with throttling
        │
        └── > 1,000,000 rows
                └── Background job with:
                    • Progress tracking
                    • Retry logic
                    • Throttling based on replication lag
                    • Ability to pause / resume
```

**Pattern: Dual-write during backfill**

When backfilling a new column, use this sequence to avoid missing data:

1. Deploy code that writes to **both** old and new columns
2. Run the backfill to populate the new column for existing rows
3. Deploy code that reads from the new column
4. Deploy code that stops writing to the old column
5. Drop the old column

---

## Rollback Strategies

### Forward-Only vs Reversible Migrations

There are two philosophies for handling migration failures:

| Approach | Description | Pros | Cons |
|---|---|---|---|
| **Reversible** | Every migration has an `up` and `down` | Can revert to any prior state | Down migrations are often untested; data loss risk |
| **Forward-only** | No down migrations; fix problems by writing new forward migrations | Simpler; no untested rollback code | Requires fast iteration to fix issues |

**Most production teams prefer forward-only migrations** for several reasons:

- Down migrations are rarely tested in CI
- Data migrations are often not reversible (e.g., you can't "un-merge" two columns)
- Rolling back a schema change while new data has been written to the new schema can cause data loss
- A quick forward fix (deploy a corrective migration) is usually faster than debugging a rollback

```
  Forward-Only Fix
  ────────────────

  V5 (buggy)           V6 (fix)
  ┌──────────────┐     ┌──────────────┐
  │ ALTER TABLE  │────▶│ ALTER TABLE  │
  │ orders ADD   │     │ orders ALTER │
  │ total INT    │     │ total TYPE   │
  │              │     │ BIGINT       │
  │ (should have │     │              │
  │  been BIGINT)│     │ (corrective  │
  └──────────────┘     │  migration)  │
                       └──────────────┘
```

### Blue-Green Database Deployments

Blue-green deployments for databases are more complex than for stateless application servers because **both environments share the same data**.

```
  Blue-Green Database Strategy
  ────────────────────────────

  Option A: Shared Database (most common)
  ────────────────────────────────────────
  ┌──────────────┐     ┌──────────────┐
  │  Blue (old)  │─┐   │  Green (new) │─┐
  │  App v1      │ │   │  App v2      │ │
  └──────────────┘ │   └──────────────┘ │
                   │                    │
                   └──────┬─────────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │   Shared DB      │
                 │   (must support  │
                 │   both v1 + v2   │
                 │   schema)        │
                 └──────────────────┘

  The schema must be compatible with BOTH versions during cutover.
  This is the expand-contract pattern applied at the deployment level.


  Option B: Separate Databases (expensive, complex)
  ──────────────────────────────────────────────────
  ┌──────────────┐         ┌──────────────┐
  │  Blue (old)  │         │  Green (new) │
  │  App v1      │         │  App v2      │
  └──────┬───────┘         └──────┬───────┘
         │                        │
         ▼                        ▼
  ┌──────────────┐         ┌──────────────┐
  │  Blue DB     │────────▶│  Green DB    │
  │  (old schema)│  sync   │  (new schema)│
  └──────────────┘         └──────────────┘

  Requires bidirectional replication or CDC (change data capture)
  to keep databases in sync during cutover. Rarely used due to
  complexity and risk of data divergence.
```

---

## Schema Evolution and Compatibility (DDIA Ch.4)

This section draws on *Designing Data-Intensive Applications* Chapter 4 (Encoding and Evolution), which describes how data formats and schemas evolve over time while maintaining compatibility between components that may be running different versions.

### Forward and Backward Compatibility

When a schema change is deployed, multiple versions of an application may be running simultaneously (rolling deployment) or different services may depend on the same database at different release cadences.

> *"In a large application you cannot usually do an instantaneous upgrade: you have to deploy new code gradually, and you may need to maintain compatibility in both directions."* — DDIA, Chapter 4

| Compatibility Type | Definition | Example |
|---|---|---|
| **Backward compatible** | New code can read data written by old code | New column is nullable — old code doesn't write it, new code handles `NULL` |
| **Forward compatible** | Old code can read data written by new code | Old code uses `SELECT *` or ignores unknown columns |

**Rules for maintaining compatibility:**

```
  Backward Compatible Changes (safe)       Breaking Changes (need expand-contract)
  ─────────────────────────────────────     ─────────────────────────────────────
  ✅ Add a nullable column                  ❌ Remove a column still in use
  ✅ Add a table                            ❌ Rename a column
  ✅ Add an index                           ❌ Change a column type
  ✅ Widen a column (VARCHAR(50) → 255)     ❌ Add NOT NULL to existing column
  ✅ Add a default value                    ❌ Change column semantics
  ✅ Relax a constraint (NOT NULL → NULL)   ❌ Tighten a constraint
```

**Applying DDIA Ch.4 principles to database migrations:**

1. **Never remove a field that readers still depend on** — this breaks backward compatibility. Deploy code first, then drop.
2. **New fields must have defaults or be nullable** — old writers won't populate new fields, so readers must handle missing values.
3. **Schema changes should be additive** — the safest evolution is always to add, never to remove or rename in one step.
4. **Use the expand-contract pattern** — this is the database equivalent of the encoding compatibility strategies Kleppmann describes for Avro, Protobuf, and Thrift.

### Encoding Formats and Schema Registries

When databases serve as integration points between services (or when event streams carry database changes via CDC), the encoding format and schema registry become critical for managing evolution.

| Format | Schema Evolution | Compatibility Enforcement | Use Case |
|---|---|---|---|
| **JSON** | Ad-hoc (no schema enforcement) | Application-level validation | REST APIs, document stores |
| **Avro** | Schema resolution at read time | Confluent Schema Registry | Kafka, data pipelines |
| **Protobuf** | Field numbers allow evolution | Buf Schema Registry | gRPC, inter-service communication |
| **Thrift** | Field IDs similar to Protobuf | Limited registry support | Legacy Facebook/Apache ecosystem |

**Avro schema evolution example:**

Avro uses a **writer schema** (used when data was written) and a **reader schema** (used when data is read). The reader can handle differences using schema resolution rules:

```json
// Writer schema (v1)
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "username", "type": "string"}
  ]
}

// Reader schema (v2) — added email with default
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "username", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
```

- **Backward compatible:** v2 reader can read v1 data (email defaults to `null`)
- **Forward compatible:** v1 reader can read v2 data (ignores the unknown `email` field)

**Schema Registry integration with CDC:**

```
  Database CDC + Schema Registry
  ──────────────────────────────

  ┌────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ Database   │────▶│ CDC          │────▶│ Schema       │────▶│ Downstream   │
  │            │     │ (Debezium)   │     │ Registry     │     │ Consumers    │
  │ Schema     │     │              │     │              │     │              │
  │ changes    │     │ Captures     │     │ Validates    │     │ Can read     │
  │ applied    │     │ row changes  │     │ schema       │     │ old + new    │
  │ via        │     │ as events    │     │ compatibility│     │ events using │
  │ migrations │     │ with schema  │     │ (backward,   │     │ schema       │
  │            │     │ metadata     │     │  forward,    │     │ resolution   │
  │            │     │              │     │  full)       │     │              │
  └────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

> **Key insight (DDIA Ch.4):** When your database is a source for event streams (via CDC or outbox pattern), schema evolution rules from encoding formats apply directly. A schema change in the database becomes a schema change in the event stream — and consumers must handle both old and new formats. Schema registries enforce compatibility rules automatically, preventing breaking changes from reaching consumers.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial database migrations documentation |
