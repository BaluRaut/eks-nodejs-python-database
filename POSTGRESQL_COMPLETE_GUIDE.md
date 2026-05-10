# PostgreSQL — Professional Complete Reference

A single-source reference for everything PostgreSQL: from first connection to enterprise production patterns. Each section answers a specific "what do I need to do" question.

---

## Table of Contents

1. [Connect & Basics](#1-connect--basics)
2. [Roles, Users & Privileges](#2-roles-users--privileges)
3. [Databases & Schemas](#3-databases--schemas)
4. [Data Types — Choosing the Right One](#4-data-types)
5. [Table Design — Professional DDL](#5-table-design)
6. [Constraints & Integrity](#6-constraints--integrity)
7. [Indexes — Every Type with Use Cases](#7-indexes)
8. [Queries — Professional Patterns](#8-queries)
9. [Partitioning](#9-partitioning)
10. [Transactions & Locking](#10-transactions--locking)
11. [Stored Functions & Triggers](#11-stored-functions--triggers)
12. [Full-Text Search](#12-full-text-search)
13. [JSONB — Document Store in PostgreSQL](#13-jsonb)
14. [Security — RLS, Column Security, Audit](#14-security)
15. [Backup & Recovery](#15-backup--recovery)
16. [Replication](#16-replication)
17. [Maintenance & Monitoring](#17-maintenance--monitoring)
18. [Performance Tuning](#18-performance-tuning)
19. [Extensions](#19-extensions)
20. [Common Production Patterns](#20-common-production-patterns)
21. [AWS RDS/Aurora PostgreSQL](#21-aws-rdsaurora-postgresql)
22. [PostgreSQL on Kubernetes](#22-postgresql-on-kubernetes)

---

## 1. Connect & Basics

```bash
# Connect via psql
psql -h hostname -p 5432 -U username -d database_name
psql "postgresql://username:password@hostname:5432/database_name"
psql "postgresql://username:password@hostname:5432/database_name?sslmode=require"

# Useful psql meta-commands
\l                        -- list databases
\c dbname                 -- switch database
\dn                       -- list schemas
\dt schema.*              -- list tables in schema
\d tablename              -- describe table (columns, indexes, constraints)
\d+ tablename             -- detailed describe
\di                       -- list indexes
\df                       -- list functions
\du                       -- list roles/users
\x                        -- toggle expanded display (wide rows)
\timing                   -- show query execution time
\e                        -- open editor for current query
\i /path/to/file.sql      -- run SQL file
\copy table FROM 'file.csv' CSV HEADER   -- import CSV
\copy (SELECT ...) TO 'out.csv' CSV HEADER -- export CSV
\q                        -- quit

# Check PostgreSQL version and connection info
SELECT version();
SELECT current_database(), current_user, inet_server_addr(), inet_server_port();
SHOW server_version;
SHOW data_directory;
```

---

## 2. Roles, Users & Privileges

> **Rule:** Never use the `postgres` superuser for applications. Create dedicated roles.

```sql
-- Create roles (a role IS a user in PostgreSQL — WITH LOGIN makes it a login role)
CREATE ROLE app_user       LOGIN PASSWORD 'strong-password-here';
CREATE ROLE readonly_user  LOGIN PASSWORD 'strong-password-here';
CREATE ROLE migrate_user   LOGIN PASSWORD 'strong-password-here' REPLICATION;
CREATE ROLE app_admin      LOGIN PASSWORD 'strong-password-here' CREATEDB;

-- Group roles (no login — used as permission bundles)
CREATE ROLE read_all  NOLOGIN;
CREATE ROLE write_all NOLOGIN;

-- Assign group roles to login roles
GRANT read_all  TO readonly_user;
GRANT write_all TO app_user;

-- Schema privileges
GRANT USAGE ON SCHEMA myapp TO app_user, readonly_user;

-- Table privileges
GRANT SELECT ON ALL TABLES IN SCHEMA myapp TO readonly_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA myapp TO app_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA myapp TO migrate_user;

-- Sequence privileges (needed for INSERT with SERIAL/BIGSERIAL)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA myapp TO app_user;

-- Default privileges (apply to future tables automatically)
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT SELECT ON TABLES TO readonly_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA myapp
    GRANT USAGE, SELECT ON SEQUENCES TO app_user;

-- Revoke
REVOKE DELETE ON orders FROM app_user;
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA myapp FROM old_user;

-- Check what a role can do
\dp tablename                             -- in psql
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'orders';

-- Change password
ALTER ROLE app_user PASSWORD 'new-strong-password';

-- Lock / unlock login
ALTER ROLE app_user NOLOGIN;
ALTER ROLE app_user LOGIN;

-- Drop role (must own nothing first)
REASSIGN OWNED BY old_user TO postgres;
DROP OWNED BY old_user;
DROP ROLE old_user;
```

---

## 3. Databases & Schemas

```sql
-- Create database
CREATE DATABASE myapp_prod
    OWNER = app_admin
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0
    CONNECTION LIMIT = 200;

-- Clone database for dev/test
CREATE DATABASE myapp_dev TEMPLATE myapp_prod;   -- only works if no active connections

-- Rename / drop
ALTER DATABASE myapp_dev RENAME TO myapp_staging;
DROP DATABASE myapp_dev;                         -- irreversible!

-- Database settings
ALTER DATABASE myapp_prod SET timezone TO 'UTC';
ALTER DATABASE myapp_prod SET search_path TO myapp, public;
ALTER DATABASE myapp_prod SET log_statement TO 'ddl';

-- Schemas (organise tables by domain)
CREATE SCHEMA myapp         AUTHORIZATION app_admin;
CREATE SCHEMA audit         AUTHORIZATION app_admin;
CREATE SCHEMA analytics     AUTHORIZATION app_admin;

-- Search path: controls which schemas are searched without prefix
SET search_path TO myapp, public;
SHOW search_path;

-- Move table between schemas
ALTER TABLE public.orders SET SCHEMA myapp;

-- Schema info
SELECT schema_name, schema_owner FROM information_schema.schemata;

-- Drop schema
DROP SCHEMA old_schema CASCADE;   -- CASCADE drops all objects inside
```

---

## 4. Data Types

### Choosing the Right Type

| Use Case | Best Type | Avoid |
|----------|-----------|-------|
| Primary key (small tables) | `SERIAL` / `INTEGER` | `VARCHAR(10)` for numeric IDs |
| Primary key (large / distributed) | `BIGSERIAL` or `UUID` | `SERIAL` (can overflow) |
| UUID as PK | `UUID` with `gen_random_uuid()` | `VARCHAR(36)` |
| Prices / money | `NUMERIC(12, 2)` | `FLOAT` (rounding errors!) |
| Large integers | `BIGINT` | `INT` for > 2 billion rows |
| Boolean | `BOOLEAN` | `SMALLINT`, `CHAR(1)` |
| Timestamps with timezone | `TIMESTAMPTZ` | `TIMESTAMP` (no TZ — ambiguous) |
| Dates only | `DATE` | `TIMESTAMPTZ` (unnecessary precision) |
| Short strings / codes | `VARCHAR(n)` | `CHAR(n)` (right-pads with spaces) |
| Long text / descriptions | `TEXT` | `VARCHAR(10000)` (TEXT is same, less typing) |
| Semi-structured data | `JSONB` | `JSON` (JSONB is indexed and faster) |
| Tags / multiple values | `TEXT[]` or `JSONB` | Multiple join tables for simple tags |
| Enumerations | `TEXT` + CHECK or `ENUM` | `SMALLINT` codes (unreadable) |
| IP addresses | `INET` | `VARCHAR(45)` |
| Ranges | `TSTZRANGE`, `INT4RANGE` | Two separate columns |
| Binary data | `BYTEA` | `TEXT` (encoding issues) |
| Monetary with currency | `NUMERIC(12, 2)` + `CHAR(3)` | `MONEY` (locale-dependent) |

```sql
-- Common type examples
CREATE TABLE type_examples (
    id              BIGSERIAL       PRIMARY KEY,
    public_id       UUID            NOT NULL DEFAULT gen_random_uuid(),
    name            TEXT            NOT NULL,
    slug            VARCHAR(100)    NOT NULL,
    status          TEXT            NOT NULL CHECK (status IN ('active','inactive','deleted')),
    score           NUMERIC(8, 4),               -- e.g. 9999.9999
    count           BIGINT          NOT NULL DEFAULT 0,
    is_active       BOOLEAN         NOT NULL DEFAULT true,
    tags            TEXT[],                       -- array of strings
    metadata        JSONB,                        -- semi-structured
    ip_address      INET,                         -- 192.168.1.1 or ::1
    valid_range     TSTZRANGE,                    -- [start, end) time range
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ                   -- NULL = not deleted
);

-- Array operations
INSERT INTO type_examples (tags) VALUES (ARRAY['postgres','aws','k8s']);
SELECT * FROM type_examples WHERE 'aws' = ANY(tags);
SELECT * FROM type_examples WHERE tags @> ARRAY['aws'];  -- contains

-- ENUM type (use sparingly — ALTER ENUM is painful)
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'cancelled', 'refunded');
ALTER TYPE order_status ADD VALUE 'processing' AFTER 'confirmed';  -- one-way only

-- Range type
INSERT INTO type_examples (valid_range) VALUES ('[2024-01-01, 2024-12-31)'::tstzrange);
SELECT * FROM type_examples WHERE valid_range @> now();  -- contains current time
```

---

## 5. Table Design

### Professional Table Template

```sql
-- Full-featured enterprise table
CREATE TABLE myapp.orders (
    -- Identity
    id              BIGSERIAL           PRIMARY KEY,
    public_id       UUID                NOT NULL DEFAULT gen_random_uuid() UNIQUE,

    -- Foreign keys
    customer_id     BIGINT              NOT NULL REFERENCES myapp.customers(id),
    tenant_id       UUID                NOT NULL,

    -- Business columns
    order_number    VARCHAR(30)         NOT NULL UNIQUE,
    status          TEXT                NOT NULL DEFAULT 'pending',
    total_amount    NUMERIC(12, 2)      NOT NULL DEFAULT 0 CHECK (total_amount >= 0),
    currency        CHAR(3)             NOT NULL DEFAULT 'USD',
    notes           TEXT,
    metadata        JSONB               NOT NULL DEFAULT '{}'::jsonb,

    -- Audit columns (on every table)
    created_at      TIMESTAMPTZ         NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ         NOT NULL DEFAULT now(),
    created_by      BIGINT              REFERENCES myapp.users(id),
    updated_by      BIGINT              REFERENCES myapp.users(id),
    deleted_at      TIMESTAMPTZ,        -- soft delete: NULL = active

    -- Table-level constraints
    CONSTRAINT orders_status_check CHECK (status IN ('pending','confirmed','shipped','cancelled')),
    CONSTRAINT orders_currency_check CHECK (currency ~ '^[A-Z]{3}$')
);

-- Comments (searchable documentation)
COMMENT ON TABLE  myapp.orders               IS 'Customer purchase orders';
COMMENT ON COLUMN myapp.orders.public_id     IS 'Exposed in API — never expose internal id';
COMMENT ON COLUMN myapp.orders.metadata      IS 'Arbitrary key-value store for extensibility';
COMMENT ON COLUMN myapp.orders.deleted_at    IS 'Soft delete — NULL means active record';

-- Auto-update updated_at trigger (attach to every mutable table)
CREATE TRIGGER orders_set_updated_at
    BEFORE UPDATE ON myapp.orders
    FOR EACH ROW EXECUTE FUNCTION myapp.set_updated_at();
```

### Table Modifications — Safe vs Unsafe

```sql
-- ✅ SAFE (no lock, instant)
ALTER TABLE orders ADD COLUMN notes TEXT;                      -- nullable, no default
ALTER TABLE orders ADD COLUMN priority INT NOT NULL DEFAULT 0; -- PG11+: no rewrite
ALTER TABLE orders ALTER COLUMN notes SET DEFAULT 'n/a';
ALTER TABLE orders ALTER COLUMN notes DROP DEFAULT;
ALTER TABLE orders DROP COLUMN IF EXISTS legacy_field;

-- ✅ SAFE (with CONCURRENTLY — does not block reads/writes)
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
DROP   INDEX CONCURRENTLY idx_orders_old;
REINDEX INDEX CONCURRENTLY idx_orders_customer;

-- ⚠️  CAREFUL (short ACCESS EXCLUSIVE lock — milliseconds if table is small)
ALTER TABLE orders ADD CONSTRAINT orders_amount_check CHECK (total_amount >= 0);
ALTER TABLE orders RENAME COLUMN old_name TO new_name;
ALTER TABLE orders ADD PRIMARY KEY (id);

-- ❌ DANGEROUS on large tables (full table rewrite — long lock)
ALTER TABLE orders ALTER COLUMN amount TYPE BIGINT;   -- use pg_repack instead
ALTER TABLE orders ADD COLUMN created_at TIMESTAMPTZ NOT NULL DEFAULT now(); -- PG10 and earlier
ALTER TABLE orders SET TABLESPACE new_tablespace;

-- Pattern: add NOT NULL safely on large tables
-- Step 1: add nullable (instant)
ALTER TABLE orders ADD COLUMN email_verified BOOLEAN DEFAULT false;
-- Step 2: backfill in batches (no lock)
DO $$ DECLARE r RECORD;
BEGIN
  FOR r IN SELECT id FROM orders WHERE email_verified IS NULL LIMIT 10000 LOOP
    UPDATE orders SET email_verified = false WHERE id = r.id;
    COMMIT;
  END LOOP;
END $$;
-- Step 3: add NOT NULL (very fast after backfill in PG12+)
ALTER TABLE orders ALTER COLUMN email_verified SET NOT NULL;
```

---

## 6. Constraints & Integrity

```sql
-- Primary key
ALTER TABLE orders ADD PRIMARY KEY (id);

-- Unique constraint (allows one NULL; use UNIQUE INDEX for multiple NULLs)
ALTER TABLE orders ADD CONSTRAINT orders_order_number_unique UNIQUE (order_number);

-- Unique partial index (unique only among active records)
CREATE UNIQUE INDEX idx_orders_active_number
    ON orders(customer_id, order_number)
    WHERE deleted_at IS NULL;

-- Foreign key with action
ALTER TABLE order_items ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id)
    ON DELETE CASCADE    -- delete items when order deleted
    ON UPDATE RESTRICT;  -- prevent changing order id

-- Check constraint
ALTER TABLE orders ADD CONSTRAINT chk_amount
    CHECK (total_amount >= 0 AND total_amount < 1000000);

-- Deferrable constraint (checked at COMMIT, not per-row — useful for circular FKs)
ALTER TABLE nodes ADD CONSTRAINT fk_parent
    FOREIGN KEY (parent_id) REFERENCES nodes(id)
    DEFERRABLE INITIALLY DEFERRED;

-- Exclusion constraint (prevent overlapping ranges — e.g. room bookings)
CREATE TABLE bookings (
    room_id  INT,
    during   TSTZRANGE,
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)
    -- prevents two bookings for same room with overlapping time ranges
);

-- NOT NULL
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;

-- List all constraints
SELECT conname, contype, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'myapp.orders'::regclass;

-- Disable constraint temporarily (for bulk loads)
ALTER TABLE order_items DISABLE TRIGGER ALL;
-- ... bulk load ...
ALTER TABLE order_items ENABLE TRIGGER ALL;
```

---

## 7. Indexes

### Complete Index Reference

```sql
-- B-TREE (default): equality, range, ORDER BY, NULLS FIRST/LAST
CREATE INDEX idx_orders_created    ON orders(created_at DESC);
CREATE INDEX idx_orders_composite  ON orders(customer_id, status, created_at DESC);

-- PARTIAL: only index rows matching a condition (smaller, faster)
CREATE INDEX idx_orders_pending    ON orders(created_at) WHERE status = 'pending';
CREATE INDEX idx_users_active      ON users(email)       WHERE deleted_at IS NULL;

-- COVERING (INCLUDE): index-only scans — no heap fetch needed
CREATE INDEX idx_orders_cover
    ON orders(customer_id, status)
    INCLUDE (total_amount, created_at, order_number);
-- Query uses this index without touching the actual table rows:
-- SELECT total_amount, created_at FROM orders WHERE customer_id=1 AND status='pending'

-- EXPRESSION: index on a computed value
CREATE INDEX idx_orders_month      ON orders(date_trunc('month', created_at));
CREATE INDEX idx_users_lower_email ON users(lower(email));  -- case-insensitive lookup

-- UNIQUE
CREATE UNIQUE INDEX idx_users_email  ON users(email) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_orders_num   ON orders(order_number);

-- GIN: JSONB, arrays, full-text search (many keys in one value)
CREATE INDEX idx_orders_metadata   ON orders USING GIN(metadata);
CREATE INDEX idx_products_tags     ON products USING GIN(tags);
CREATE INDEX idx_articles_fts      ON articles USING GIN(to_tsvector('english', title || ' ' || body));
-- JSONB path operator index
CREATE INDEX idx_meta_status       ON orders USING GIN((metadata -> 'status'));

-- GiST: geometric/PostGIS, range types, similarity
CREATE INDEX idx_locations_geo     ON locations USING GIST(coordinates);
CREATE INDEX idx_bookings_range    ON bookings  USING GIST(during);

-- BRIN: huge append-only tables ordered by column (e.g. logs, events by time)
-- 1000x smaller than B-tree but only efficient when data is physically ordered
CREATE INDEX idx_events_brin       ON events USING BRIN(created_at) WITH (pages_per_range = 128);

-- HASH: equality-only, faster than B-tree for exact match (rarely needed)
CREATE INDEX idx_sessions_token    ON sessions USING HASH(token);

-- Concurrent build (no table lock — use in production)
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);

-- Index maintenance
REINDEX INDEX CONCURRENTLY idx_orders_customer;   -- rebuild bloated index
DROP   INDEX CONCURRENTLY idx_orders_old;

-- Analyse index usage
SELECT indexname, idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY idx_scan DESC;

-- Find unused indexes (safe to drop after confirming)
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pk_%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (sequential scans on large tables)
SELECT relname, seq_scan, seq_tup_read,
       idx_scan, pg_size_pretty(pg_total_relation_size(relid)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
  AND pg_total_relation_size(relid) > 100 * 1024 * 1024  -- > 100 MB
ORDER BY seq_scan DESC;
```

---

## 8. Queries

### Professional Query Patterns

```sql
-- UPSERT (INSERT ... ON CONFLICT)
INSERT INTO users (email, name, updated_at)
VALUES ('alice@example.com', 'Alice', now())
ON CONFLICT (email)
DO UPDATE SET
    name       = EXCLUDED.name,
    updated_at = now()
WHERE users.name IS DISTINCT FROM EXCLUDED.name;  -- only update if actually changed

-- UPSERT and return the resulting row
INSERT INTO sessions (user_id, token, expires_at)
VALUES (1, gen_random_uuid(), now() + interval '1 hour')
ON CONFLICT (user_id) DO UPDATE SET
    token      = EXCLUDED.token,
    expires_at = EXCLUDED.expires_at
RETURNING id, token, expires_at;

-- Soft delete and filter
SELECT * FROM orders WHERE deleted_at IS NULL;
UPDATE orders SET deleted_at = now(), updated_by = 123 WHERE id = 456;

-- CTEs (Common Table Expressions)
WITH ranked_orders AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
    FROM orders WHERE deleted_at IS NULL
),
top_customers AS (
    SELECT customer_id, COUNT(*) AS order_count, SUM(total_amount) AS lifetime_value
    FROM orders WHERE status = 'confirmed' AND deleted_at IS NULL
    GROUP BY customer_id
    HAVING COUNT(*) >= 5
)
SELECT o.*, tc.lifetime_value
FROM ranked_orders o
JOIN top_customers tc USING (customer_id)
WHERE o.rn = 1;   -- only most recent order per customer

-- Window functions
SELECT
    id, customer_id, total_amount, created_at,
    SUM(total_amount)   OVER (PARTITION BY customer_id ORDER BY created_at) AS cumulative_spend,
    AVG(total_amount)   OVER (PARTITION BY customer_id) AS avg_order_value,
    RANK()              OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS rank_by_amount,
    LAG(total_amount)   OVER (PARTITION BY customer_id ORDER BY created_at) AS prev_order_amount,
    LEAD(total_amount)  OVER (PARTITION BY customer_id ORDER BY created_at) AS next_order_amount,
    FIRST_VALUE(id)     OVER (PARTITION BY customer_id ORDER BY created_at) AS first_order_id
FROM orders
WHERE deleted_at IS NULL;

-- LATERAL JOIN (correlated subquery as join — can reference left-side row)
SELECT c.id, c.name, recent.order_count, recent.last_order_at
FROM customers c
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS order_count, MAX(created_at) AS last_order_at
    FROM orders o WHERE o.customer_id = c.id
) recent ON true;

-- SELECT FOR UPDATE SKIP LOCKED (queue pattern)
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY priority DESC, created_at ASC
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- Batch delete (no lock on full table)
DELETE FROM events
WHERE id IN (
    SELECT id FROM events WHERE created_at < now() - interval '90 days'
    LIMIT 1000
);

-- UPDATE with JOIN (UPDATE ... FROM)
UPDATE orders o SET
    status = 'cancelled',
    updated_at = now()
FROM customers c
WHERE o.customer_id = c.id
  AND c.account_status = 'suspended'
  AND o.status = 'pending'
RETURNING o.id, c.email;

-- Recursive CTE (hierarchical data — org charts, category trees)
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth, name::TEXT AS path
    FROM categories WHERE parent_id IS NULL    -- root nodes

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;

-- COPY for bulk import (fastest way to load data)
COPY orders(customer_id, order_number, total_amount, status)
FROM '/tmp/orders.csv' CSV HEADER;

-- COPY to export
COPY (SELECT * FROM orders WHERE created_at > '2024-01-01') TO '/tmp/export.csv' CSV HEADER;

-- Explain plan (always use ANALYZE + BUFFERS in dev — never in prod on slow queries)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending';

-- Understand plan output:
-- Seq Scan       → no index used (check if index needed)
-- Index Scan     → uses index but fetches heap (ok)
-- Index Only Scan → uses covering index (best — no heap fetch)
-- Bitmap Scan    → multiple index conditions combined (ok for range queries)
-- Hash Join      → good for large set joins
-- Nested Loop    → good for small inner sets with index
-- cost=X..Y      → estimated cost (X=startup, Y=total)
-- actual time    → real milliseconds
-- Buffers hit    → from cache; read → from disk (high read = cache miss)
```

---

## 9. Partitioning

```sql
-- RANGE partitioning — most common for time-series
CREATE TABLE events (
    id          BIGSERIAL,
    type        TEXT        NOT NULL,
    payload     JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create partitions (automate with pg_partman extension)
CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Indexes on parent apply to all partitions
CREATE INDEX ON events(type);
CREATE INDEX ON events(created_at);

-- LIST partitioning — by known set of values
CREATE TABLE orders_by_region (
    id      BIGSERIAL, region TEXT, amount NUMERIC
) PARTITION BY LIST (region);

CREATE TABLE orders_apac  PARTITION OF orders_by_region FOR VALUES IN ('AU','JP','SG','IN');
CREATE TABLE orders_emea  PARTITION OF orders_by_region FOR VALUES IN ('GB','DE','FR','NL');
CREATE TABLE orders_amer  PARTITION OF orders_by_region FOR VALUES IN ('US','CA','BR','MX');
CREATE TABLE orders_other PARTITION OF orders_by_region DEFAULT;

-- HASH partitioning — even distribution by key
CREATE TABLE order_items (id BIGSERIAL, order_id BIGINT, sku TEXT)
PARTITION BY HASH (order_id);

CREATE TABLE order_items_0 PARTITION OF order_items FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE order_items_1 PARTITION OF order_items FOR VALUES WITH (MODULUS 8, REMAINDER 1);
-- ... repeat for 2-7

-- SUB-PARTITIONING (range then hash)
CREATE TABLE metrics (ts TIMESTAMPTZ, host TEXT, value NUMERIC)
PARTITION BY RANGE (ts);

CREATE TABLE metrics_2024_01 PARTITION OF metrics
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    PARTITION BY HASH (host);    -- sub-partition by host

CREATE TABLE metrics_2024_01_h0 PARTITION OF metrics_2024_01
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

-- Partition management
-- Drop old partition (instant — no DELETE scan)
DROP TABLE events_2022_q1;

-- Detach and archive (data preserved, just detached from parent)
ALTER TABLE events DETACH PARTITION events_2022_q1;
-- Move to archive schema or pg_dump separately

-- Attach pre-populated table as new partition
CREATE TABLE events_2025_q1 (LIKE events INCLUDING ALL);
-- Populate events_2025_q1...
ALTER TABLE events ATTACH PARTITION events_2025_q1
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Verify partition pruning
EXPLAIN SELECT * FROM events WHERE created_at >= '2024-04-01' AND created_at < '2024-07-01';
-- Should show only events_2024_q2 in plan, not all partitions

-- List partitions
SELECT parent.relname AS parent, child.relname AS partition,
       pg_get_expr(child.relpartbound, child.oid) AS bounds,
       pg_size_pretty(pg_total_relation_size(child.oid)) AS size
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child  ON pg_inherits.inhrelid  = child.oid
WHERE parent.relname = 'events'
ORDER BY child.relname;
```

---

## 10. Transactions & Locking

```sql
-- Basic transaction
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- or ROLLBACK; to undo

-- Savepoints (partial rollback inside transaction)
BEGIN;
    INSERT INTO orders VALUES (DEFAULT, 1, 'pending', 100);
    SAVEPOINT before_items;
    INSERT INTO order_items VALUES (DEFAULT, currval('orders_id_seq'), 'sku-1');
    -- if this fails:
    ROLLBACK TO SAVEPOINT before_items;
    -- order still exists, just without items
COMMIT;

-- Transaction isolation levels
BEGIN ISOLATION LEVEL READ COMMITTED;    -- default: sees committed rows at each statement
BEGIN ISOLATION LEVEL REPEATABLE READ;  -- snapshot at BEGIN: no phantom reads
BEGIN ISOLATION LEVEL SERIALIZABLE;     -- full serializability: detects write-write conflicts

-- Advisory locks (application-level mutex — no table needed)
SELECT pg_advisory_lock(12345);         -- exclusive lock, blocks until acquired
SELECT pg_advisory_xact_lock(12345);    -- released automatically at END
SELECT pg_try_advisory_lock(12345);     -- non-blocking: returns true/false
SELECT pg_advisory_unlock(12345);

-- Row-level locks
SELECT * FROM orders WHERE id = 1 FOR UPDATE;           -- exclusive row lock
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;    -- fail immediately if locked
SELECT * FROM orders WHERE id = 1 FOR UPDATE SKIP LOCKED; -- skip locked rows (queue)
SELECT * FROM orders WHERE id = 1 FOR SHARE;            -- shared lock (allow other reads)

-- Deadlock prevention: always lock rows in the same order
-- Bad:  Tx1 locks row A then B; Tx2 locks row B then A → deadlock
-- Good: both lock lowest ID first

-- Lock monitoring
SELECT
    pid, pg_blocking_pids(pid) AS blocked_by,
    query, state, wait_event_type, wait_event,
    now() - query_start AS duration
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Show all current locks
SELECT l.pid, l.locktype, l.relation::regclass, l.mode, l.granted,
       a.query, a.state
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
ORDER BY l.granted, l.pid;
```

---

## 11. Stored Functions & Triggers

```sql
-- Utility function: auto-update updated_at
CREATE OR REPLACE FUNCTION myapp.set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$;

-- Apply to every table
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON myapp.orders
    FOR EACH ROW EXECUTE FUNCTION myapp.set_updated_at();

-- Audit trigger: record every change in audit table
CREATE TABLE audit.order_changes (
    id          BIGSERIAL PRIMARY KEY,
    table_name  TEXT,
    operation   TEXT,           -- INSERT UPDATE DELETE
    old_data    JSONB,
    new_data    JSONB,
    changed_by  TEXT DEFAULT current_user,
    changed_at  TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit.log_changes()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO audit.order_changes(table_name, operation, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP = 'INSERT' THEN NULL ELSE to_jsonb(OLD) END,
        CASE WHEN TG_OP = 'DELETE' THEN NULL ELSE to_jsonb(NEW) END
    );
    RETURN NULL;  -- AFTER trigger: return value ignored for row triggers
END;
$$;

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON myapp.orders
    FOR EACH ROW EXECUTE FUNCTION audit.log_changes();

-- Function: paginated query helper
CREATE OR REPLACE FUNCTION myapp.get_orders(
    p_customer_id  BIGINT,
    p_status       TEXT DEFAULT NULL,
    p_limit        INT  DEFAULT 20,
    p_offset       INT  DEFAULT 0
)
RETURNS TABLE (
    id           BIGINT,
    order_number TEXT,
    status       TEXT,
    total_amount NUMERIC,
    created_at   TIMESTAMPTZ,
    total_count  BIGINT
)
LANGUAGE sql STABLE AS $$
    SELECT
        id, order_number, status, total_amount, created_at,
        COUNT(*) OVER () AS total_count
    FROM myapp.orders
    WHERE customer_id = p_customer_id
      AND deleted_at IS NULL
      AND (p_status IS NULL OR status = p_status)
    ORDER BY created_at DESC
    LIMIT p_limit OFFSET p_offset;
$$;

-- Call function
SELECT * FROM myapp.get_orders(123, 'pending', 10, 0);

-- Procedure (can COMMIT inside — PG11+)
CREATE OR REPLACE PROCEDURE myapp.archive_old_orders(cutoff_date TIMESTAMPTZ)
LANGUAGE plpgsql AS $$
DECLARE
    batch_size INT := 1000;
    deleted_count INT;
BEGIN
    LOOP
        DELETE FROM myapp.orders
        WHERE id IN (
            SELECT id FROM myapp.orders
            WHERE created_at < cutoff_date AND status = 'cancelled'
            LIMIT batch_size
        );
        GET DIAGNOSTICS deleted_count = ROW_COUNT;
        EXIT WHEN deleted_count = 0;
        COMMIT;  -- release locks between batches
        PERFORM pg_sleep(0.1);  -- be nice to production
    END LOOP;
END;
$$;

CALL myapp.archive_old_orders('2023-01-01'::TIMESTAMPTZ);
```

---

## 12. Full-Text Search

```sql
-- Setup: tsvector column for documents
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;

-- Populate (English stemming + weighting: A=title, B=body)
UPDATE articles SET search_vector =
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B');

-- Auto-update via trigger
CREATE OR REPLACE FUNCTION articles_search_vector_update()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.body, '')), 'B');
    RETURN NEW;
END;
$$;
CREATE TRIGGER articles_search_update
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION articles_search_vector_update();

-- GIN index for fast FTS
CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);

-- Search queries
-- Simple: find rows containing 'kubernetes'
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'kubernetes') query
WHERE search_vector @@ query
ORDER BY rank DESC LIMIT 20;

-- AND: must contain both words
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'kubernetes & deployment');

-- OR: either word
WHERE search_vector @@ to_tsquery('english', 'kubernetes | docker');

-- NOT: exclude word
WHERE search_vector @@ to_tsquery('english', 'kubernetes & !deprecated');

-- Phrase search (adjacent words)
WHERE search_vector @@ phraseto_tsquery('english', 'rolling update');

-- Prefix search (for autocomplete)
WHERE search_vector @@ to_tsquery('english', 'kubern:*');

-- Highlight matches in results
SELECT title,
       ts_headline('english', body, to_tsquery('english', 'kubernetes'),
           'MaxWords=30, MinWords=10, StartSel=<mark>, StopSel=</mark>') AS snippet
FROM articles
WHERE search_vector @@ to_tsquery('english', 'kubernetes');
```

---

## 13. JSONB

```sql
-- JSONB operators
-- -> returns JSON
-- ->> returns TEXT
-- #> path as JSON, #>> path as text

SELECT
    metadata -> 'shipping'             AS shipping_json,     -- {"city":"NY","zip":"10001"}
    metadata ->> 'status'              AS status_text,       -- "confirmed"
    metadata -> 'tags' -> 0            AS first_tag,         -- "priority"
    metadata #> '{address,city}'       AS city_json,
    metadata #>> '{address,city}'      AS city_text;         -- "New York"

-- Filter by JSONB value
SELECT * FROM orders WHERE metadata ->> 'channel' = 'mobile';
SELECT * FROM orders WHERE metadata @> '{"status": "vip"}'::jsonb;  -- contains
SELECT * FROM orders WHERE metadata ? 'discount_code';              -- key exists
SELECT * FROM orders WHERE metadata ?| ARRAY['discount','coupon'];  -- any key exists
SELECT * FROM orders WHERE metadata ?& ARRAY['channel','source'];   -- all keys exist

-- Index for JSONB containment (@>) and key existence (?)
CREATE INDEX idx_orders_metadata ON orders USING GIN(metadata);

-- Index on specific path (more selective, smaller)
CREATE INDEX idx_orders_channel ON orders ((metadata ->> 'channel'));

-- Update JSONB (immutable — must replace)
UPDATE orders SET metadata = metadata || '{"priority": "high"}'::jsonb WHERE id = 1;
UPDATE orders SET metadata = metadata - 'old_key';           -- remove key
UPDATE orders SET metadata = jsonb_set(metadata, '{shipping,city}', '"Boston"');

-- Build JSONB from columns
SELECT jsonb_build_object(
    'id', id,
    'customer', jsonb_build_object('id', customer_id, 'name', customer_name),
    'items', jsonb_agg(jsonb_build_object('sku', sku, 'qty', quantity))
) AS order_json
FROM orders JOIN order_items USING (id)
GROUP BY id, customer_id, customer_name;

-- Expand JSONB array to rows
SELECT id, jsonb_array_elements(metadata -> 'tags') AS tag
FROM orders WHERE metadata ? 'tags';

-- Expand JSONB object to key-value pairs
SELECT id, key, value
FROM orders, jsonb_each_text(metadata);

-- Aggregate rows into JSONB
SELECT customer_id, jsonb_agg(to_jsonb(orders)) AS all_orders
FROM orders GROUP BY customer_id;
```

---

## 14. Security

### Row-Level Security (RLS)

```sql
-- Enable RLS on table
ALTER TABLE myapp.orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE myapp.orders FORCE ROW LEVEL SECURITY;  -- also applies to table owner

-- Tenant isolation policy (SaaS multi-tenant)
CREATE POLICY orders_tenant_isolation ON myapp.orders
    USING      (tenant_id = current_setting('app.tenant_id')::UUID)  -- for SELECT
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::UUID); -- for INSERT/UPDATE

-- User-level policy (users see only their own orders)
CREATE POLICY orders_owner_policy ON myapp.orders
    USING (customer_id = current_setting('app.user_id')::BIGINT);

-- Admin bypass policy
CREATE POLICY orders_admin_policy ON myapp.orders
    TO app_admin                         -- only this role
    USING (true)                         -- all rows
    WITH CHECK (true);

-- Set tenant context (done by app/middleware at connection start)
SET app.tenant_id = 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee';

-- RLS does NOT apply to superuser — use SET ROLE for testing
SET ROLE app_user;
SELECT * FROM myapp.orders;  -- only sees rows for current tenant
RESET ROLE;

-- Column-level security: grant access to specific columns only
GRANT SELECT(id, name, email) ON myapp.users TO public_api_user;
-- NOT GRANTED: password_hash, ssn, credit_card

-- Audit view: who changed what
CREATE VIEW audit.recent_changes AS
SELECT * FROM audit.order_changes ORDER BY changed_at DESC LIMIT 1000;
GRANT SELECT ON audit.recent_changes TO compliance_team;
```

### SSL & Authentication (pg_hba.conf)

```bash
# /etc/postgresql/16/main/pg_hba.conf
# TYPE  DATABASE  USER         ADDRESS         METHOD
local   all       postgres                     peer        # local unix socket
local   all       all                          scram-sha-256
host    all       all          127.0.0.1/32    scram-sha-256
host    all       all          10.0.0.0/8      scram-sha-256   # VPC range
hostssl all       app_user     0.0.0.0/0       scram-sha-256   # SSL required
```

```sql
-- Enforce SSL for a user
ALTER ROLE app_user SET ssl TO on;

-- Check SSL connections
SELECT ssl, client_addr, usename, datname
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid);

-- Log all DDL and connection events
ALTER DATABASE myapp_prod SET log_connections    TO on;
ALTER DATABASE myapp_prod SET log_disconnections TO on;
ALTER DATABASE myapp_prod SET log_statement      TO 'ddl';  -- ddl | mod | all | none
```

---

## 15. Backup & Recovery

```bash
# ─── pg_dump ──────────────────────────────────────────────
# Custom format (compressed, parallelizable — recommended)
pg_dump -h host -U user -Fc -f backup.dump dbname

# Directory format (parallel dump with -j)
pg_dump -h host -U user -Fd -j 8 -f backup_dir/ dbname

# Plain SQL (human-readable — large files, not parallelizable)
pg_dump -h host -U user -f backup.sql dbname

# Schema only (DDL, no data)
pg_dump --schema-only -f schema.sql dbname

# Data only
pg_dump --data-only -Fc -f data.dump dbname

# Specific schema
pg_dump -n myapp -Fc -f myapp_schema.dump dbname

# Specific table
pg_dump -t myapp.orders -Fc -f orders.dump dbname

# Exclude table data (schema + all tables except logs)
pg_dump --exclude-table-data='audit.*' -Fc -f without_audit.dump dbname

# ─── pg_restore ───────────────────────────────────────────
# Full restore (parallel)
pg_restore -h host -U user -d dbname -j 8 --no-owner --no-acl backup.dump

# Restore single table into existing database
pg_restore -h host -U user -d dbname -t orders --no-owner backup.dump

# List contents without restoring
pg_restore --list backup.dump

# Restore only specific items from list
pg_restore --list backup.dump > restore.list
# Edit restore.list to keep only desired items
pg_restore -L restore.list -d dbname backup.dump

# ─── Continuous WAL archiving (self-managed) ──────────────
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-wal-archive/%f'
restore_command = 'aws s3 cp s3://my-wal-archive/%f %p'

# Base backup for PITR
pg_basebackup -h host -U replication_user -D /backup/base -Ft -z -P --wal-method=stream

# Recover to specific point in time
# recovery.conf (PG11 and earlier) / postgresql.conf (PG12+)
restore_command = 'aws s3 cp s3://my-wal-archive/%f %p'
recovery_target_time = '2024-03-15 14:30:00+00'
recovery_target_action = 'promote'
```

---

## 16. Replication

### Streaming Replication (Physical — HA)

```bash
# Primary: postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB
synchronous_commit = on      # for synchronous replication

# Primary: create replication user
CREATE ROLE replicator REPLICATION LOGIN PASSWORD 'rep-password';

# Primary: pg_hba.conf
host replication replicator 10.0.1.0/24 scram-sha-256

# Replica: create base backup
pg_basebackup -h primary_host -U replicator -D /var/lib/postgresql/data -P --wal-method=stream -R

# -R flag writes standby.signal and adds primary_conninfo to postgresql.conf automatically

# Replica: postgresql.conf (auto-written by pg_basebackup -R)
primary_conninfo = 'host=primary_host port=5432 user=replicator password=rep-password'

# Monitor lag
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

### Logical Replication (Table-Level)

```sql
-- Publisher (source)
ALTER SYSTEM SET wal_level = 'logical';
SELECT pg_reload_conf();

CREATE PUBLICATION my_pub FOR TABLE orders, customers;  -- specific tables
CREATE PUBLICATION all_pub FOR ALL TABLES;              -- all tables

-- Subscriber (target — can be a different PG version or cloud DB)
CREATE SUBSCRIPTION my_sub
CONNECTION 'host=source_host dbname=myapp user=replicator password=pass'
PUBLICATION my_pub;

-- Monitor
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_replication_slots;

-- Pause / resume
ALTER SUBSCRIPTION my_sub DISABLE;
ALTER SUBSCRIPTION my_sub ENABLE;

-- Used for: zero-downtime major version upgrades, partial replication, cross-cloud migration
```

---

## 17. Maintenance & Monitoring

```sql
-- ─── Vacuum ───────────────────────────────────────────────
-- Auto-vacuum runs automatically, but you may trigger manually:
VACUUM ANALYZE orders;             -- reclaim space + update stats
VACUUM VERBOSE ANALYZE orders;     -- verbose output
VACUUM FULL orders;                -- full rewrite — use pg_repack instead (no lock)

-- Tune auto-vacuum for high-write tables
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor  = 0.01,  -- trigger at 1% dead tuples (default 20%)
    autovacuum_analyze_scale_factor = 0.005, -- analyze at 0.5%
    autovacuum_vacuum_cost_delay    = 2,     -- ms delay between pages (reduce I/O impact)
    autovacuum_vacuum_threshold     = 50     -- minimum rows before triggering
);

-- Check dead tuple accumulation
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- ─── Table & Index Sizes ──────────────────────────────────
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename))       AS table_only,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename))        AS indexes
FROM pg_tables
WHERE schemaname = 'myapp'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- ─── Active Connections ───────────────────────────────────
SELECT datname, usename, client_addr, state,
       count(*) AS connections,
       max(now() - query_start) AS max_duration
FROM pg_stat_activity
WHERE state != 'idle'
GROUP BY 1,2,3,4
ORDER BY connections DESC;

-- Check max_connections vs current
SELECT count(*) AS active,
       (SELECT setting::INT FROM pg_settings WHERE name = 'max_connections') AS max
FROM pg_stat_activity;

-- ─── Bloat Detection ──────────────────────────────────────
-- Table bloat (dead space as % of total)
SELECT tablename,
       round(100 * n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS bloat_pct,
       pg_size_pretty(pg_total_relation_size('myapp.'||tablename)) AS size
FROM pg_stat_user_tables
WHERE schemaname = 'myapp'
  AND (n_live_tup + n_dead_tup) > 10000
ORDER BY bloat_pct DESC NULLS LAST;

-- ─── Long-Running Queries ────────────────────────────────
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '30 seconds'
ORDER BY duration DESC;

-- Kill query (soft)
SELECT pg_cancel_backend(pid);     -- sends SIGINT — cancels query, keeps connection
-- Kill connection (hard)
SELECT pg_terminate_backend(pid);  -- sends SIGTERM — disconnects client

-- ─── Replication Lag ─────────────────────────────────────
SELECT
    client_addr,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag
FROM pg_stat_replication;
```

---

## 18. Performance Tuning

### postgresql.conf — Production Settings

```ini
# Memory (tune to instance size)
shared_buffers             = 8GB       # 25% of RAM (e.g. on 32GB instance)
effective_cache_size       = 24GB      # 75% of RAM — tells planner about OS cache
work_mem                   = 32MB      # per sort/hash; multiply by max_connections!
maintenance_work_mem       = 2GB       # for VACUUM, CREATE INDEX, CLUSTER
huge_pages                 = try       # try to use huge pages (Linux)

# WAL & Checkpoints
wal_buffers                = 64MB
min_wal_size               = 1GB
max_wal_size               = 8GB
checkpoint_completion_target = 0.9    # spread checkpoint writes over 90% of interval
wal_compression            = on       # compress WAL records

# Connections
max_connections            = 100      # keep low — use PgBouncer
connection_timeout         = 10

# Planner
random_page_cost           = 1.1      # SSD: 1.1–1.5 (default 4.0 is for HDD)
effective_io_concurrency   = 200      # SSD: 200+ (HDD: 2)
enable_parallel_query      = on
max_parallel_workers_per_gather = 4
max_parallel_workers       = 8
max_worker_processes       = 16

# Logging
log_min_duration_statement = 1000     # log queries slower than 1 second
log_lock_waits             = on
log_temp_files             = 64MB     # log queries spilling to disk
log_autovacuum_min_duration = 250     # log slow autovacuums
log_checkpoints            = on
log_connections            = off      # too noisy with PgBouncer
log_line_prefix            = '%t [%p] %u@%d '

# Statistics
track_io_timing            = on       # show I/O in EXPLAIN ANALYZE
track_functions            = all
```

### pg_stat_statements — Find Slow Queries

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 by total time
SELECT round(total_exec_time::numeric, 1) AS total_ms,
       calls,
       round(mean_exec_time::numeric, 2)  AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       rows / calls                        AS avg_rows,
       query
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top by average time (slowest individual queries)
SELECT round(mean_exec_time::numeric, 2) AS avg_ms, calls, query
FROM pg_stat_statements
WHERE calls > 100
ORDER BY mean_exec_time DESC LIMIT 10;

-- Top by I/O (cache misses)
SELECT query,
       shared_blks_hit + shared_blks_read AS total_blks,
       round(100.0 * shared_blks_hit /
           NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 0
ORDER BY shared_blks_read DESC LIMIT 10;

-- Cache hit rate (should be > 99% for OLTP)
SELECT
    sum(blks_hit) * 100.0 / NULLIF(sum(blks_hit) + sum(blks_read), 0) AS cache_hit_pct
FROM pg_stat_database WHERE datname = current_database();

SELECT pg_stat_statements_reset();  -- reset stats
```

---

## 19. Extensions

```sql
-- List installed extensions
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions WHERE installed_version IS NOT NULL;

-- Install / drop
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
DROP EXTENSION IF EXISTS old_extension;
```

| Extension | Use Case | Install |
|-----------|----------|---------|
| `pg_stat_statements` | Query performance profiling | Bundled |
| `pg_trgm` | Fuzzy text search, LIKE index | Bundled |
| `pgcrypto` | `gen_random_uuid()`, `crypt()`, `digest()` | Bundled |
| `uuid-ossp` | Alternative UUID generation | Bundled |
| `hstore` | Key-value in column (legacy — use JSONB) | Bundled |
| `tablefunc` | `crosstab()` pivot queries | Bundled |
| `postgres_fdw` | Foreign data wrapper (query remote PG) | Bundled |
| `pg_partman` | Automatic partition creation/maintenance | External |
| `pg_repack` | Rebuild bloated tables online (no lock) | External |
| `PostGIS` | Geographic/spatial queries, GPS data | External |
| `pgvector` | Vector similarity search (AI embeddings) | External |
| `TimescaleDB` | Time-series optimizations on top of PG | External |
| `Citus` | Horizontal sharding across nodes | External |
| `pg_cron` | Cron-style scheduled jobs inside DB | External |
| `pgtap` | Unit testing for SQL functions | External |
| `pg_logical` | Logical replication with more control | External |

```sql
-- pg_trgm: fuzzy LIKE search with index
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%kuberentes%';  -- typo tolerant!
SELECT * FROM products WHERE similarity(name, 'kubernetes') > 0.3;

-- pgvector: AI semantic search
CREATE EXTENSION vector;
ALTER TABLE documents ADD COLUMN embedding vector(1536);  -- OpenAI ada-002 dimensions
CREATE INDEX ON documents USING ivfflat(embedding vector_cosine_ops) WITH (lists = 100);
SELECT id, content, 1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector LIMIT 10;

-- pg_cron: schedule inside the database
CREATE EXTENSION pg_cron;
SELECT cron.schedule('nightly-vacuum', '0 2 * * *', 'VACUUM ANALYZE orders');
SELECT cron.schedule('purge-sessions', '*/5 * * * *',
    'DELETE FROM sessions WHERE expires_at < now()');
SELECT * FROM cron.job;
```

---

## 20. Common Production Patterns

### Queue with SKIP LOCKED

```sql
-- Job queue table
CREATE TABLE job_queue (
    id          BIGSERIAL   PRIMARY KEY,
    type        TEXT        NOT NULL,
    payload     JSONB       NOT NULL,
    status      TEXT        NOT NULL DEFAULT 'pending',
    attempts    INT         NOT NULL DEFAULT 0,
    max_retries INT         NOT NULL DEFAULT 3,
    run_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at  TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    error       TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_queue_pickup ON job_queue(run_at)
    WHERE status IN ('pending', 'retry');

-- Worker: pick up one job atomically (no two workers get same job)
WITH picked AS (
    SELECT id FROM job_queue
    WHERE status IN ('pending', 'retry')
      AND run_at <= now()
      AND attempts < max_retries
    ORDER BY run_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue SET
    status     = 'processing',
    started_at = now(),
    attempts   = attempts + 1
WHERE id = (SELECT id FROM picked)
RETURNING *;

-- Complete job
UPDATE job_queue SET status = 'completed', completed_at = now() WHERE id = $1;

-- Fail job (with retry)
UPDATE job_queue SET
    status = CASE WHEN attempts >= max_retries THEN 'failed' ELSE 'retry' END,
    error  = $2,
    run_at = now() + (2 ^ attempts * interval '1 minute')  -- exponential backoff
WHERE id = $1;
```

### Soft Delete Pattern

```sql
-- All queries must filter deleted_at IS NULL
-- Use a view to hide the filter
CREATE VIEW myapp.active_orders AS
SELECT * FROM myapp.orders WHERE deleted_at IS NULL;

-- Make view updatable (allow INSERT/UPDATE/DELETE on view)
CREATE RULE active_orders_insert AS
    ON INSERT TO myapp.active_orders DO INSTEAD
    INSERT INTO myapp.orders VALUES (NEW.*);

-- Or use security barrier view (prevents filter bypass)
CREATE VIEW myapp.active_orders WITH (security_barrier = true) AS
SELECT * FROM myapp.orders WHERE deleted_at IS NULL;

-- Soft delete function
CREATE OR REPLACE FUNCTION myapp.soft_delete(p_table TEXT, p_id BIGINT)
RETURNS VOID LANGUAGE plpgsql AS $$
BEGIN
    EXECUTE format('UPDATE myapp.%I SET deleted_at = now() WHERE id = $1', p_table)
    USING p_id;
END;
$$;
```

### Pagination — Cursor-Based (Better than OFFSET)

```sql
-- OFFSET pagination (bad: gets slower as offset grows)
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;  -- scans 10020 rows!

-- Cursor-based pagination (fast: always uses index)
-- First page
SELECT * FROM orders
WHERE deleted_at IS NULL
ORDER BY id ASC LIMIT 20;
-- Returns last id = 500

-- Next page (pass last_id from previous page)
SELECT * FROM orders
WHERE deleted_at IS NULL AND id > 500   -- last_id from previous page
ORDER BY id ASC LIMIT 20;
-- Always O(log n) regardless of page number

-- Multi-column cursor (when sorting by non-unique column)
-- First page
SELECT * FROM orders WHERE deleted_at IS NULL
ORDER BY created_at DESC, id DESC LIMIT 20;
-- Returns: last row created_at='2024-03-15 10:00', id=999

-- Next page (both columns needed for tie-breaking)
SELECT * FROM orders
WHERE deleted_at IS NULL
  AND (created_at, id) < ('2024-03-15 10:00', 999)  -- row comparison
ORDER BY created_at DESC, id DESC LIMIT 20;
```

### Connection String Best Practices

```bash
# Always specify sslmode in production
postgresql://app_user:pass@host:5432/myapp?sslmode=require

# Read from replica for SELECT-heavy operations
postgresql://app_user:pass@replica.host:5432/myapp?sslmode=require&target_session_attrs=read-only

# Connection through PgBouncer
postgresql://app_user:pass@pgbouncer:5432/myapp?sslmode=require

# Node.js (pg library)
const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  ssl: { rejectUnauthorized: true },
  max: 10,                    // pool size (keep low — use PgBouncer for many pods)
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  statement_timeout: 30000,   // kill queries running > 30s
});
```

---

## 21. AWS RDS/Aurora PostgreSQL

### Parameter Group — Key Settings

```bash
# Create custom parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name myapp-pg16 \
  --db-parameter-group-family postgres16 \
  --description "Production PostgreSQL 16 settings"

# Apply parameters
aws rds modify-db-parameter-group \
  --db-parameter-group-name myapp-pg16 \
  --parameters \
    "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot" \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=log_lock_waits,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=work_mem,ParameterValue=33554432,ApplyMethod=immediate"

# Attach to RDS instance
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-parameter-group-name myapp-pg16 \
  --apply-immediately
```

### RDS vs Aurora PostgreSQL — Key Differences

| Feature | RDS PostgreSQL | Aurora PostgreSQL |
|---------|---------------|------------------|
| Storage | EBS (gp3/io2) — per GB + IOPS | Shared distributed storage — pay per GB used |
| Storage scaling | Manual resize | Automatic (up to 128 TB) |
| Read replicas | Up to 5, async | Up to 15, shared storage (near-zero lag) |
| Failover | 20–120s (Multi-AZ) | 20–60s (data already on replica storage) |
| Backtrack | No | Yes (Aurora MySQL only) |
| Cloning | Snapshot restore (slow) | Fast clone (copy-on-write, instant) |
| Global DB | No | Yes (cross-region, 1s lag) |
| Serverless | No | Aurora Serverless v2 |
| Cost | Lower for constant workloads | Higher base, better at scale |
| Extensions | 85+ available | ~85, slightly different list |

### RDS Limitations vs Self-Managed PostgreSQL

```sql
-- NOT available on RDS (managed by AWS):
-- - superuser access
-- - pg_hba.conf direct edit (use Parameter Group rds.pg_hba.conf_method)
-- - archive_command (AWS manages WAL archiving)
-- - COPY TO/FROM local filesystem (use aws_s3 extension instead)
-- - postgres_fdw to non-RDS servers (allowed but requires security group)

-- RDS-specific: load data from S3
SELECT aws_s3.table_import_from_s3(
    'orders',
    '',
    '(FORMAT CSV, HEADER true)',
    'my-bucket', 'data/orders.csv', 'us-east-1'
);

-- RDS-specific: export to S3
SELECT aws_s3.query_export_to_s3(
    'SELECT * FROM orders WHERE created_at > ''2024-01-01''',
    'my-bucket', 'exports/orders-2024.csv', 'us-east-1',
    options := '(FORMAT CSV, HEADER true)'
);

-- Enhanced monitoring (OS-level metrics in 1-second intervals)
-- Enable via Console or:
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --monitoring-interval 5 \
  --monitoring-role-arn arn:aws:iam::ACCOUNT:role/rds-monitoring-role

-- IAM database authentication (no password needed — uses IAM token)
-- Create RDS auth token (valid 15 minutes)
TOKEN=$(aws rds generate-db-auth-token \
  --hostname mydb.rds.amazonaws.com \
  --port 5432 \
  --username app_user \
  --region us-east-1)
PGPASSWORD=$TOKEN psql -h mydb.rds.amazonaws.com -U app_user myapp
```

---

## 22. PostgreSQL on Kubernetes

### Quick Commands

```bash
# Connect to K8s PostgreSQL pod
kubectl exec -it postgres-0 -n databases -- psql -U postgres myapp

# Run SQL from file against K8s pod
kubectl exec -i postgres-0 -n databases -- psql -U postgres myapp < migration.sql

# Check replication status in Zalando Patroni cluster
kubectl exec -it postgres-0 -n databases -- patronictl -c /home/postgres/postgres.yml list

# Scale read replicas
kubectl scale statefulset postgres --replicas=3 -n databases

# Port-forward for local access
kubectl port-forward svc/postgres 5432:5432 -n databases

# Watch pod status during rolling restart
kubectl get pods -n databases -w

# Backup manually
kubectl exec -it postgres-0 -n databases -- \
  pg_dump -Fc myapp > backup-$(date +%Y%m%d).dump

# Check PVC usage
kubectl exec -it postgres-0 -n databases -- df -h /var/lib/postgresql/data

# Expand PVC size (StorageClass must have allowVolumeExpansion: true)
kubectl patch pvc postgres-storage-postgres-0 -n databases \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

### Zalando Postgres Operator — Manage Clusters

```bash
# Install Postgres Operator
helm install postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator --create-namespace

# Create cluster
kubectl apply -f postgres-cluster.yaml

# Get cluster status
kubectl get postgresql -n databases
kubectl describe postgresql prod-postgres-cluster -n databases

# Get auto-generated credentials
kubectl get secret prod-postgres-cluster.app-user.credentials.postgresql.acid.zalan.do \
  -n databases -o jsonpath='{.data.password}' | base64 -d

# Failover manually (for maintenance)
kubectl exec -it prod-postgres-cluster-0 -n databases -- \
  patronictl -c /home/postgres/postgres.yml failover prod-postgres-cluster

# Rolling restart (no downtime with HA setup)
kubectl exec -it prod-postgres-cluster-0 -n databases -- \
  patronictl -c /home/postgres/postgres.yml restart prod-postgres-cluster
```

---

## Quick Reference Card

```sql
-- Most used one-liners for daily work

-- What's running right now?
SELECT pid, now()-query_start AS age, state, left(query,80) FROM pg_stat_activity WHERE state != 'idle';

-- What's blocked?
SELECT pid, pg_blocking_pids(pid), left(query,80) FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid))>0;

-- Top 5 biggest tables?
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 5;

-- Cache hit rate?
SELECT round(100*sum(blks_hit)/NULLIF(sum(blks_hit)+sum(blks_read),0),2) AS pct FROM pg_stat_database WHERE datname=current_database();

-- Which tables need vacuum?
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables WHERE n_dead_tup > 10000 ORDER BY n_dead_tup DESC;

-- Active connections by state?
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- Replica lag?
SELECT client_addr, pg_size_pretty(pg_wal_lsn_diff(sent_lsn,replay_lsn)) AS lag FROM pg_stat_replication;

-- What are my slow queries?
SELECT round(mean_exec_time::numeric,1) AS avg_ms, calls, left(query,80) FROM pg_stat_statements WHERE calls>50 ORDER BY mean_exec_time DESC LIMIT 10;
```
