# Database Complete Reference Guide

Enterprise-grade reference covering all major database engines, AWS managed services, Kubernetes deployment patterns, table management, migration, backup, and disaster recovery.

---

## Table of Contents

1. [AWS Database Services — Which to Use When](#1-aws-database-services)
2. [PostgreSQL — Enterprise Table & Schema Management](#2-postgresql-enterprise-management)
3. [MySQL — Enterprise Table & Schema Management](#3-mysql-enterprise-management)
4. [Aurora — Features Beyond Standard RDS](#4-aurora-features)
5. [Kubernetes Database Deployment Patterns](#5-kubernetes-database-patterns)
6. [Index Strategies — PostgreSQL & MySQL](#6-index-strategies)
7. [Partitioning — Range, List, Hash](#7-partitioning)
8. [Backup & Restore — All Methods](#8-backup-and-restore)
9. [Disaster Recovery — RPO/RTO Matrix](#9-disaster-recovery)
10. [Database Migration — All Paths](#10-database-migration)
11. [Connection Pooling & Management](#11-connection-pooling)
12. [Performance Tuning & Monitoring](#12-performance-tuning)
13. [Multi-Tenant Database Patterns](#13-multi-tenant-patterns)
14. [Enterprise Database Feature Comparison](#14-enterprise-comparison)

---

## 1. AWS Database Services

### Complete AWS Database Portfolio

| Service | Engine | Best For | Max Size |
|---------|--------|----------|----------|
| **RDS PostgreSQL** | PostgreSQL 13–16 | OLTP, general purpose, familiar tooling | 64 TB |
| **RDS MySQL** | MySQL 8.0 | LAMP stack, WordPress, legacy apps | 64 TB |
| **RDS MariaDB** | MariaDB 10.x | MySQL-compatible, Galera clustering | 64 TB |
| **RDS Oracle** | Oracle EE/SE2 | Oracle-specific features, APEX | 64 TB |
| **RDS SQL Server** | SQL Server 2019/2022 | .NET apps, Windows ecosystem | 16 TB |
| **Aurora PostgreSQL** | PostgreSQL-compatible | High performance OLTP, global apps | 128 TB |
| **Aurora MySQL** | MySQL-compatible | High performance OLTP, read-heavy | 128 TB |
| **Aurora Serverless v2** | PostgreSQL or MySQL | Variable / unpredictable workloads | 128 TB |
| **DynamoDB** | Key-value / Document | High throughput, infinite scale, simple access patterns | Unlimited |
| **ElastiCache Redis** | Redis 7.x | Session, cache, pub/sub, leaderboards | 340 GB per node |
| **ElastiCache Memcached** | Memcached 1.6 | Simple cache, multi-thread | 345 GB per node |
| **Redshift** | Columnar (Postgres-compatible) | Data warehouse, analytics, BI tools | Petabytes |
| **DocumentDB** | MongoDB 4.0/5.0 compatible | Document storage, MongoDB migration | 64 TB |
| **Keyspaces** | Apache Cassandra compatible | Time series, wide column, Cassandra migration | Unlimited |
| **Neptune** | Graph (Gremlin / SPARQL) | Social networks, fraud detection, knowledge graphs | 64 TB |
| **QLDB** | Ledger (immutable journal) | Financial records, supply chain, audit trails | Unlimited |
| **Timestream** | Time-series | IoT metrics, DevOps telemetry | Unlimited |
| **MemoryDB for Redis** | Redis | Redis with multi-AZ durable persistence | 128 TB |

### Decision Tree

```
Is data structured / relational?
├─ YES → Need MySQL compatibility? → YES → Aurora MySQL or RDS MySQL
│         Need PostgreSQL features? → YES → Aurora PostgreSQL or RDS PostgreSQL
│         Need Oracle/SQL Server? → YES → RDS Oracle / RDS SQL Server
│         Variable load? → YES → Aurora Serverless v2
│
├─ NO → Key-value / simple lookup? → YES → DynamoDB
        Documents (JSON)? → YES → DocumentDB
        Graph relationships? → YES → Neptune
        Time series / IoT? → YES → Timestream
        Cache / session? → YES → ElastiCache Redis
        Analytics / BI? → YES → Redshift
        Audit / immutable? → YES → QLDB
```

---

## 2. PostgreSQL — Enterprise Management

### Database & Schema Hierarchy

```
PostgreSQL Cluster (1 port 5432)
└── Database: myapp_prod
    ├── Schema: public (default)
    ├── Schema: orders (domain separation)
    ├── Schema: analytics (read-only views)
    └── Schema: audit (triggers, history)
```

```sql
-- Create isolated schema per domain
CREATE SCHEMA orders AUTHORIZATION app_user;
CREATE SCHEMA analytics AUTHORIZATION readonly_user;

-- Search path (equivalent of USE DATABASE in MySQL)
SET search_path TO orders, public;

-- Grant access
GRANT USAGE ON SCHEMA orders TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA orders TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA orders GRANT SELECT ON TABLES TO readonly_user;
```

### Table Management — Full Example

```sql
-- Enterprise table with all best practices
CREATE TABLE orders.orders (
    id              BIGSERIAL PRIMARY KEY,
    order_number    VARCHAR(20)  NOT NULL UNIQUE,
    customer_id     BIGINT       NOT NULL REFERENCES customers(id),
    tenant_id       UUID         NOT NULL DEFAULT gen_random_uuid(),
    status          VARCHAR(20)  NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','confirmed','shipped','cancelled')),
    total_amount    NUMERIC(12,2) NOT NULL CHECK (total_amount >= 0),
    currency        CHAR(3)      NOT NULL DEFAULT 'USD',
    metadata        JSONB,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,                        -- soft delete
    CONSTRAINT orders_amount_currency_check CHECK (currency ~ '^[A-Z]{3}$')
) PARTITION BY RANGE (created_at);                      -- range partitioned by date

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_updated_at
BEFORE UPDATE ON orders.orders
FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Enable Row-Level Security
ALTER TABLE orders.orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders.orders
    USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

### Table Alteration — Zero-Downtime Patterns

```sql
-- SAFE: Add nullable column (no table rewrite)
ALTER TABLE orders ADD COLUMN notes TEXT;

-- SAFE: Add column with DEFAULT (PostgreSQL 11+ uses metadata, no rewrite)
ALTER TABLE orders ADD COLUMN priority INT NOT NULL DEFAULT 0;

-- SAFE: Add index concurrently (does not lock table)
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);
CREATE INDEX CONCURRENTLY idx_orders_status_created ON orders(status, created_at DESC);
CREATE INDEX CONCURRENTLY idx_orders_metadata ON orders USING GIN(metadata);

-- SAFE: Drop index concurrently
DROP INDEX CONCURRENTLY idx_old_index;

-- CAREFUL: Add NOT NULL constraint (requires backfill first)
-- Step 1: Add nullable
ALTER TABLE orders ADD COLUMN email_verified BOOLEAN DEFAULT false;
-- Step 2: Backfill in batches
UPDATE orders SET email_verified = false WHERE email_verified IS NULL;
-- Step 3: Add NOT NULL (fast after backfill)
ALTER TABLE orders ALTER COLUMN email_verified SET NOT NULL;

-- CAREFUL: Rename column (use views for backward compat)
ALTER TABLE orders RENAME COLUMN old_name TO new_name;

-- DANGEROUS: avoid on large tables without CONCURRENTLY
-- ALTER TABLE orders ALTER COLUMN amount TYPE BIGINT; -- full table rewrite!
```

### PostgreSQL Maintenance

```sql
-- Vacuum to reclaim dead tuple space
VACUUM ANALYZE orders;
VACUUM VERBOSE ANALYZE orders;     -- verbose output

-- Auto-vacuum tuning for high-write tables
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- vacuum at 1% dead tuples (default 20%)
    autovacuum_analyze_scale_factor = 0.005, -- analyze at 0.5%
    autovacuum_vacuum_cost_delay = 2         -- ms delay (reduce I/O impact)
);

-- Check table bloat
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    n_dead_tup, n_live_tup,
    round(100 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND query_start < now() - interval '5 minutes'
ORDER BY duration DESC;

-- Kill long-running query
SELECT pg_cancel_backend(pid);   -- soft cancel
SELECT pg_terminate_backend(pid); -- hard terminate

-- Lock monitoring
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks l JOIN pg_stat_activity a USING (pid)
WHERE NOT granted;
```

---

## 3. MySQL — Enterprise Management

### Database Structure

```sql
-- MySQL databases (equivalent to PostgreSQL schemas)
CREATE DATABASE myapp_orders CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE DATABASE myapp_analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE myapp_orders;
```

### Table Management

```sql
-- Enterprise MySQL table
CREATE TABLE orders (
    id              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    order_number    VARCHAR(20)     NOT NULL,
    customer_id     BIGINT UNSIGNED NOT NULL,
    tenant_id       CHAR(36)        NOT NULL,
    status          ENUM('pending','confirmed','shipped','cancelled') NOT NULL DEFAULT 'pending',
    total_amount    DECIMAL(12,2)   NOT NULL,
    currency        CHAR(3)         NOT NULL DEFAULT 'USD',
    metadata        JSON,
    created_at      DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at      DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
    deleted_at      DATETIME(6),
    PRIMARY KEY (id),
    UNIQUE KEY uq_order_number (order_number),
    KEY idx_customer_status (customer_id, status),
    KEY idx_tenant_created (tenant_id, created_at),
    CONSTRAINT chk_amount CHECK (total_amount >= 0),
    CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
) ENGINE=InnoDB
  ROW_FORMAT=DYNAMIC
  PARTITION BY RANGE (YEAR(created_at)) (
      PARTITION p2023 VALUES LESS THAN (2024),
      PARTITION p2024 VALUES LESS THAN (2025),
      PARTITION p2025 VALUES LESS THAN (2026),
      PARTITION pmax  VALUES LESS THAN MAXVALUE
  );
```

### MySQL Zero-Downtime Alterations

```sql
-- Use gh-ost or pt-online-schema-change for large tables
-- gh-ost: no triggers, uses binary log
-- pt-osc: uses triggers

-- gh-ost example (run from CLI):
-- gh-ost \
--   --host=mydb.rds.amazonaws.com \
--   --database=myapp_orders \
--   --table=orders \
--   --alter="ADD COLUMN notes TEXT" \
--   --execute

-- Online DDL (MySQL 8.0 with InnoDB)
ALTER TABLE orders
    ADD COLUMN notes TEXT,
    ALGORITHM=INPLACE, LOCK=NONE;   -- non-blocking

-- Add index online
ALTER TABLE orders
    ADD INDEX idx_status_created (status, created_at),
    ALGORITHM=INPLACE, LOCK=NONE;

-- Check if operation is online
SHOW VARIABLES LIKE 'innodb_online_alter_log_max_size';
```

### MySQL Maintenance

```sql
-- Table statistics
ANALYZE TABLE orders;

-- Check table
CHECK TABLE orders EXTENDED;

-- Optimize (rebuilds + defragments — use with caution on large tables)
OPTIMIZE TABLE orders;

-- Slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;       -- log queries > 1 second
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Show processlist
SHOW FULL PROCESSLIST;

-- Kill query
KILL QUERY 12345;
KILL CONNECTION 12345;

-- InnoDB status
SHOW ENGINE INNODB STATUS\G

-- Table size
SELECT
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb,
    table_rows
FROM information_schema.TABLES
WHERE table_schema = 'myapp_orders'
ORDER BY (data_length + index_length) DESC;
```

---

## 4. Aurora Features

### Aurora PostgreSQL — Key Differentiators

```sql
-- Aurora Parallel Query (analytical queries on OLTP data)
-- Enabled per-session
SET aurora_pq_force = ON;

-- Backtrack (unique to Aurora MySQL — point-in-time rewind without restore)
-- Rewind entire cluster to a point in time within backtrack window
-- aws rds backtrack-db-cluster --db-cluster-identifier mycluster --backtrack-to "2024-01-15T10:00:00Z"

-- Aurora global write forwarding (Aurora PostgreSQL)
-- Read replicas can accept writes, forwarded to writer
SET aurora.read_replica_allow_writes = ON;  -- on read replica

-- Babelfish for Aurora PostgreSQL (SQL Server TDS protocol compatibility)
-- Connect SQL Server apps to Aurora PostgreSQL without code changes

-- Aurora Machine Learning (invoke SageMaker / Comprehend from SQL)
SELECT aws_comprehend.detect_sentiment(review_text, 'en') AS sentiment
FROM product_reviews;
```

### Aurora-Specific Monitoring

```sql
-- Aurora replica lag
SELECT server_id, durable_lsn, highest_lsn_rcvd,
       (highest_lsn_rcvd - durable_lsn) AS replica_lag_lsn
FROM aurora_replica_status()
WHERE session_id = 'MASTER_SESSION_ID';

-- Aurora storage volume info
SELECT * FROM aurora_volume_logical_start_lsn();

-- Fast cloning (copy-on-write — instant regardless of DB size)
-- aws rds restore-db-cluster-to-point-in-time \
--   --source-db-cluster-identifier prod-cluster \
--   --db-cluster-identifier dev-clone \
--   --restore-type copy-on-write \
--   --use-latest-restorable-time

-- Performance Insights (via Console or API)
-- Queries by load, wait events, top SQL
```

---

## 5. Kubernetes Database Patterns

### StatefulSet — Running PostgreSQL in K8s

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres-headless
  replicas: 1           # scale via Zalando Postgres Operator for HA
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      securityContext:
        runAsUser: 999
        fsGroup: 999
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: postgres-config
          mountPath: /etc/postgresql/postgresql.conf
          subPath: postgresql.conf
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2"
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3-encrypted    # custom StorageClass
      resources:
        requests:
          storage: 100Gi
```

### StorageClass — Optimized for Databases

```yaml
# storage-class-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "6000"            # up to 16000 IOPS
  throughput: "500"       # MB/s
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:ACCOUNT:key/KEY-ID"
reclaimPolicy: Retain     # CRITICAL: Retain PV when PVC deleted
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Zalando Postgres Operator — Production HA

```yaml
# postgres-cluster.yaml (Zalando Operator)
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: prod-postgres-cluster
  namespace: databases
spec:
  teamId: "myteam"
  volume:
    size: 200Gi
    storageClass: gp3-encrypted
  numberOfInstances: 3         # 1 primary + 2 replicas
  postgresql:
    version: "16"
    parameters:
      max_connections: "200"
      shared_buffers: "2GB"
      work_mem: "16MB"
      maintenance_work_mem: "256MB"
      wal_level: "replica"
      max_wal_senders: "10"
  patroni:
    initdb:
      encoding: "UTF8"
      locale: "en_US.UTF-8"
  resources:
    requests:
      cpu: 1
      memory: 4Gi
    limits:
      cpu: 4
      memory: 8Gi
  users:
    app_user:
      - superuser
    readonly_user: []
  databases:
    myapp: app_user
  enableLogicalBackup: true
  logicalBackupSchedule: "0 2 * * *"   # 2am daily
```

### CronJob — Automated Backup to S3

```yaml
# postgres-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: databases
spec:
  schedule: "0 1 * * *"     # 1am daily
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: backup-sa   # IRSA role for S3 access
          containers:
          - name: backup
            image: postgres:16-alpine
            command:
            - /bin/sh
            - -c
            - |
              FILENAME="postgres-$(date +%Y%m%d-%H%M%S).dump"
              pg_dump \
                -h postgres-headless.databases.svc.cluster.local \
                -U $PGUSER \
                -Fc \                          # custom format (compressed)
                -f /tmp/$FILENAME \
                myapp
              aws s3 cp /tmp/$FILENAME s3://my-db-backups/postgres/$FILENAME
              echo "Backup $FILENAME uploaded to S3"
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGUSER
              value: app_user
```

### External Secrets Operator — Sync AWS Secrets to K8s

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: myapp
spec:
  refreshInterval: 1m           # sync every minute
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret             # K8s secret name
    creationPolicy: Owner
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: prod/myapp/db
      property: password
  - secretKey: DB_HOST
    remoteRef:
      key: prod/myapp/db
      property: host
```

---

## 6. Index Strategies

### PostgreSQL Index Types

```sql
-- B-Tree (default): equality, range, ORDER BY
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_orders_status_customer ON orders(status, customer_id);

-- Partial index: index a subset of rows (much smaller, faster)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';      -- only indexes pending rows

-- Expression index: index on function result
CREATE INDEX idx_orders_month ON orders(date_trunc('month', created_at));

-- Covering index: include non-key columns to avoid heap fetch
CREATE INDEX idx_orders_cover ON orders(customer_id, status)
INCLUDE (total_amount, created_at);    -- index-only scan possible

-- GIN: for JSONB, arrays, full-text search
CREATE INDEX idx_orders_metadata ON orders USING GIN(metadata);
CREATE INDEX idx_products_tags ON products USING GIN(tags);      -- array column
CREATE INDEX idx_docs_search ON documents USING GIN(to_tsvector('english', content));

-- GiST: geometric data, range types, PostGIS
CREATE INDEX idx_locations_geo ON locations USING GIST(coordinates);

-- BRIN: very large tables with naturally ordered data (write-once logs)
CREATE INDEX idx_events_brin ON events USING BRIN(created_at);   -- 10x smaller than B-tree

-- Composite index rules
-- Index (a, b, c) serves queries on:  a | a,b | a,b,c
-- Does NOT serve: b | c | b,c (without a)

-- Check index usage
SELECT indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY idx_scan DESC;

-- Find unused indexes
SELECT indexname FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_%';

-- Explain + analyze
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending';
```

### MySQL Index Strategies

```sql
-- Composite index — leading column rule
CREATE INDEX idx_orders_composite ON orders (status, customer_id, created_at);
-- Serves: status | status+customer_id | status+customer_id+created_at
-- Does NOT serve: customer_id alone

-- Covering index
CREATE INDEX idx_orders_cover ON orders (customer_id, status, total_amount);

-- Prefix index for long VARCHAR
CREATE INDEX idx_email_prefix ON users (email(20));  -- first 20 chars

-- Full-text index
CREATE FULLTEXT INDEX idx_content ON articles(title, body);
SELECT * FROM articles WHERE MATCH(title,body) AGAINST ('kubernetes' IN BOOLEAN MODE);

-- JSON index (MySQL 8.0)
ALTER TABLE orders ADD COLUMN metadata_status VARCHAR(20)
    GENERATED ALWAYS AS (metadata->>'$.status') STORED;
CREATE INDEX idx_metadata_status ON orders(metadata_status);

-- Check index usage
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'myapp_orders' AND object_name = 'orders';

-- Show query execution plan
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 123;
```

---

## 7. Partitioning

### PostgreSQL Table Partitioning

```sql
-- RANGE partitioning by month
CREATE TABLE events (
    id          BIGSERIAL,
    event_type  VARCHAR(50),
    payload     JSONB,
    created_at  TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Default partition catches anything outside defined ranges
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Indexes on partitioned table (applies to all partitions)
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);

-- LIST partitioning by region
CREATE TABLE orders_by_region (
    id BIGSERIAL, region VARCHAR(10), amount NUMERIC
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders_by_region FOR VALUES IN ('US', 'CA');
CREATE TABLE orders_eu PARTITION OF orders_by_region FOR VALUES IN ('UK', 'DE', 'FR');

-- HASH partitioning by customer_id (even distribution)
CREATE TABLE order_items (
    id BIGSERIAL, order_id BIGINT, product_id BIGINT
) PARTITION BY HASH (order_id);

CREATE TABLE order_items_p0 PARTITION OF order_items FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE order_items_p1 PARTITION OF order_items FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE order_items_p2 PARTITION OF order_items FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE order_items_p3 PARTITION OF order_items FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Partition pruning — verify optimizer uses partitions
EXPLAIN SELECT * FROM events WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
-- Should show: Append with only 1 partition selected

-- Drop old partition (instant — no DELETE overhead)
DROP TABLE events_2022_01;

-- Attach existing table as new partition
CREATE TABLE events_2025_01 (LIKE events INCLUDING ALL);
ALTER TABLE events ATTACH PARTITION events_2025_01
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

---

## 8. Backup & Restore — All Methods

### AWS RDS/Aurora Automated Backup

```bash
# View automated snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier mydb \
  --snapshot-type automated

# Point-in-time restore (up to 5 minutes ago)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mydb \
  --target-db-instance-identifier mydb-restored \
  --restore-time 2024-03-15T10:30:00Z

# Manual snapshot (retained even if DB deleted)
aws rds create-db-snapshot \
  --db-instance-identifier mydb \
  --db-snapshot-identifier mydb-before-migration

# Copy snapshot cross-region (for DR)
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:ACCOUNT:snapshot:mydb-snap \
  --target-db-snapshot-identifier mydb-snap-eu \
  --region eu-west-1 \
  --kms-key-id arn:aws:kms:eu-west-1:ACCOUNT:key/KEY

# Copy snapshot cross-account
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier mydb-snap \
  --attribute-name restore \
  --values-to-add "TARGET_ACCOUNT_ID"
```

### pg_dump / pg_restore — PostgreSQL

```bash
# Full database backup (custom format — compressed, parallelizable)
pg_dump \
  -h mydb.rds.amazonaws.com \
  -U myuser \
  -Fc \
  --no-acl \
  --no-owner \
  -f backup.dump \
  myapp_db

# Parallel dump (8 jobs) into directory format
pg_dump \
  -h mydb.rds.amazonaws.com \
  -U myuser \
  -Fd \
  -j 8 \
  -f backup_dir \
  myapp_db

# Backup specific schema only
pg_dump -n orders myapp_db -Fc -f orders_schema.dump

# Backup specific table only
pg_dump -t orders.orders myapp_db -Fc -f orders_table.dump

# Restore full database
pg_restore \
  -h target.rds.amazonaws.com \
  -U myuser \
  -d myapp_db \
  -j 8 \
  --no-owner \
  --no-acl \
  backup.dump

# Restore single table
pg_restore -h target.rds.amazonaws.com -U myuser -d myapp_db -t orders backup.dump

# Schema-only dump (DDL no data)
pg_dump --schema-only -f schema.sql myapp_db

# Data-only dump
pg_dump --data-only -Fc -f data.dump myapp_db

# Continuous WAL archiving (base backup + WAL streaming)
pg_basebackup \
  -h mydb.rds.amazonaws.com \
  -U replication_user \
  -D /backup/base \
  -Ft -z -P \
  --wal-method=stream
```

### mysqldump / MySQL backup

```bash
# Full database backup
mysqldump \
  -h mydb.rds.amazonaws.com \
  -u myuser -p \
  --single-transaction \      # consistent snapshot without locking
  --routines \                # include stored procedures
  --triggers \
  --events \
  --hex-blob \
  myapp_db > backup.sql

# Parallel backup with mydumper (much faster than mysqldump)
mydumper \
  -h mydb.rds.amazonaws.com \
  -u myuser -p password \
  -B myapp_db \
  -o /backup/ \
  -t 8 \                      # 8 threads
  --compress

# Restore with myloader
myloader \
  -h target.rds.amazonaws.com \
  -u myuser -p password \
  -B myapp_db \
  -d /backup/ \
  -t 8

# Backup specific tables
mysqldump myapp_db orders customers --single-transaction > partial.sql

# Schema only
mysqldump --no-data myapp_db > schema.sql
```

### AWS Backup — Centralized Policy

```bash
# Create backup plan via AWS Backup (covers RDS, Aurora, EBS, EFS, DynamoDB)
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "production-db-backup",
  "Rules": [
    {
      "RuleName": "daily-backup",
      "TargetBackupVaultName": "prod-vault",
      "ScheduleExpression": "cron(0 1 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 120,
      "Lifecycle": {
        "MoveToColdStorageAfterDays": 30,
        "DeleteAfterDays": 365
      },
      "CopyActions": [
        {
          "DestinationBackupVaultArn": "arn:aws:backup:eu-west-1:ACCOUNT:backup-vault:dr-vault",
          "Lifecycle": {"DeleteAfterDays": 90}
        }
      ]
    }
  ]
}'
```

---

## 9. Disaster Recovery

### RPO / RTO Comparison Matrix

| Strategy | RPO (Data Loss) | RTO (Downtime) | Cost | Complexity |
|---------|----------------|----------------|------|------------|
| RDS automated backup + PITR | 5 minutes | 30–60 min | Low | Low |
| RDS Read Replica promote | 0–30 sec lag | 1–5 min | Medium | Medium |
| RDS Multi-AZ | 0 (sync) | 20–120 sec | Medium | Low |
| Aurora Multi-AZ | 0 (shared storage) | 20–60 sec | Medium | Low |
| Aurora Global Database | ~1 sec | < 1 min | High | Medium |
| Aurora + Multi-Region warm standby | ~1 sec | < 5 min | High | High |
| Active-Active (DynamoDB Global) | ~0 | 0 (automatic) | Very High | Very High |

### RDS Multi-AZ vs Read Replica

| Feature | Multi-AZ | Read Replica |
|---------|---------|-------------|
| Purpose | High availability (HA) | Read scaling + DR |
| Replication | Synchronous | Asynchronous |
| Promotes automatically | Yes | No (manual) |
| Readable | No (standby not accessible) | Yes |
| Same region | Yes | Same or cross-region |
| Lag | 0 (sync) | Seconds to minutes |

### Aurora Disaster Recovery Tiers

```bash
# Tier 1: Aurora Multi-AZ (default — 6 copies across 3 AZs)
# Built-in, no configuration needed. Failover ~30 seconds.

# Tier 2: Aurora Global Database cross-region
aws rds create-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:ACCOUNT:cluster:prod

# Add secondary region
aws rds create-db-cluster \
  --db-cluster-identifier prod-eu-replica \
  --global-cluster-identifier my-global-cluster \
  --engine aurora-postgresql \
  --region eu-west-1

# Tier 3: Failover (manual or automated)
aws rds failover-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --target-db-cluster-identifier prod-eu-replica
```

---

## 10. Database Migration

### Migration Path Matrix

| Source | Target | Tool | Approach |
|--------|--------|------|----------|
| On-prem PostgreSQL | RDS PostgreSQL | pg_dump + restore | Offline (downtime) |
| On-prem PostgreSQL | RDS PostgreSQL | pglogical / pg_logical | Online (CDC) |
| On-prem PostgreSQL | Aurora PostgreSQL | AWS DMS | Online |
| MySQL 5.7 | Aurora MySQL | Snapshot + restore | Offline |
| MySQL 5.7 | Aurora MySQL | AWS DMS | Online |
| Oracle | Aurora PostgreSQL | AWS SCT + DMS | Online |
| SQL Server | Aurora PostgreSQL | AWS SCT + DMS | Online |
| MongoDB | DocumentDB | mongodump / mongorestore | Offline |
| Cassandra | Amazon Keyspaces | AWS DMS (beta) | Online |
| Self-managed K8s DB | RDS | pg_dump / mysqldump | Offline |
| RDS MySQL | Aurora MySQL | Aurora snapshot restore | Near-zero downtime |

### AWS DMS — Online Migration Setup

```bash
# 1. Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier my-dms-instance \
  --replication-instance-class dms.r5.xlarge \
  --allocated-storage 100 \
  --multi-az \
  --engine-version 3.5.2

# 2. Create source endpoint
aws dms create-endpoint \
  --endpoint-identifier source-postgres \
  --endpoint-type source \
  --engine-name postgres \
  --server-name on-prem-db.company.com \
  --port 5432 \
  --database-name myapp \
  --username myuser \
  --password mypass

# 3. Create target endpoint
aws dms create-endpoint \
  --endpoint-identifier target-aurora \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --server-name aurora.cluster-xxx.rds.amazonaws.com \
  --port 5432 \
  --database-name myapp \
  --username admin \
  --password targetpass

# 4. Create replication task (full load + CDC)
aws dms create-replication-task \
  --replication-task-identifier migrate-myapp \
  --source-endpoint-arn arn:aws:dms:...:endpoint:source \
  --target-endpoint-arn arn:aws:dms:...:endpoint:target \
  --replication-instance-arn arn:aws:dms:...:rep:instance \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [
      {"rule-type":"selection","rule-id":"1","rule-name":"include-all",
       "object-locator":{"schema-name":"public","table-name":"%"},
       "rule-action":"include"},
      {"rule-type":"transformation","rule-id":"2","rule-name":"lowercase-schema",
       "rule-action":"convert-lowercase",
       "rule-target":"schema",
       "object-locator":{"schema-name":"%"}}
    ]
  }'

# 5. Monitor task progress
aws dms describe-replication-tasks \
  --filters Name=replication-task-id,Values=migrate-myapp
```

### Flyway — K8s Migration Init Container

```yaml
# deployment-with-migration.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
      - name: db-migrate
        image: flyway/flyway:10-alpine
        args:
          - -url=jdbc:postgresql://$(DB_HOST):5432/$(DB_NAME)
          - -user=$(DB_USER)
          - -password=$(DB_PASS)
          - -locations=filesystem:/flyway/sql
          - -outOfOrder=false
          - -validateOnMigrate=true
          - migrate
        env:
        - name: DB_HOST
          valueFrom: {secretKeyRef: {name: db-secret, key: host}}
        - name: DB_NAME
          value: myapp
        - name: DB_USER
          valueFrom: {secretKeyRef: {name: db-secret, key: username}}
        - name: DB_PASS
          valueFrom: {secretKeyRef: {name: db-secret, key: password}}
        volumeMounts:
        - name: sql-migrations
          mountPath: /flyway/sql
      containers:
      - name: app
        image: myapp:latest
        # App starts only after migrations succeed
      volumes:
      - name: sql-migrations
        configMap:
          name: flyway-migrations
```

### Flyway Migration Files

```sql
-- V1__create_orders_schema.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- V2__add_total_amount.sql
ALTER TABLE orders ADD COLUMN total_amount NUMERIC(12,2);

-- V3__add_order_index.sql
CREATE INDEX CONCURRENTLY idx_orders_customer ON orders(customer_id);

-- R__refresh_summary_view.sql  (repeatable migration — runs when checksum changes)
CREATE OR REPLACE VIEW order_summary AS
SELECT customer_id, count(*) AS order_count, sum(total_amount) AS total_spent
FROM orders GROUP BY customer_id;
```

---

## 11. Connection Pooling

### PgBouncer Configuration

```ini
# pgbouncer.ini
[databases]
myapp = host=aurora.cluster.rds.amazonaws.com port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction          # most efficient: connection released after each txn
max_client_conn = 1000           # max app connections to pgbouncer
default_pool_size = 20           # connections pgbouncer holds to postgres
min_pool_size = 5
reserve_pool_size = 5            # emergency pool for slow clients
reserve_pool_timeout = 3
server_idle_timeout = 600        # close idle server connections after 10min
client_idle_timeout = 0
log_connections = 0
log_disconnections = 0
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
listen_addr = 0.0.0.0
listen_port = 5432
```

### RDS Proxy — Serverless Connection Pooling

```bash
# RDS Proxy: AWS-managed, IAM auth, handles Lambda/serverless spiky connections
aws rds create-db-proxy \
  --db-proxy-name myapp-proxy \
  --engine-family POSTGRESQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:...","IAMAuth":"REQUIRED"}]' \
  --role-arn arn:aws:iam::ACCOUNT:role/rds-proxy-role \
  --vpc-subnet-ids subnet-aaa subnet-bbb \
  --vpc-security-group-ids sg-xxx

# Register DB with proxy
aws rds register-db-proxy-targets \
  --db-proxy-name myapp-proxy \
  --db-cluster-identifiers my-aurora-cluster
```

---

## 12. Performance Tuning & Monitoring

### PostgreSQL Key Parameters

```sql
-- postgresql.conf key settings for production RDS/Aurora
-- (set via Parameter Group in AWS)

-- Memory
shared_buffers = 25% of RAM           -- e.g. 8GB for 32GB instance
effective_cache_size = 75% of RAM     -- hint to planner
work_mem = 16MB                        -- per sort/hash operation (multiply by connections!)
maintenance_work_mem = 1GB             -- for VACUUM, CREATE INDEX

-- WAL
wal_level = replica
max_wal_size = 4GB
checkpoint_completion_target = 0.9    -- spread checkpoint I/O
wal_buffers = 64MB

-- Connections
max_connections = 200                  -- lower + use PgBouncer
connection_timeout = 10

-- Query planner
enable_partition_pruning = on
enable_parallel_query = on
max_parallel_workers_per_gather = 4
random_page_cost = 1.1                 -- for SSDs (default 4.0 is for spinning disk)
effective_io_concurrency = 200         -- for SSDs

-- Logging
log_slow_statements = 500             -- ms, log slow queries
log_lock_waits = on
log_temp_files = 64MB
```

### Performance Insights (AWS)

```bash
# Get top SQL by average active sessions
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-XXXXXXXXXXX \
  --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.sql_tokenized","Limit":10}}]' \
  --start-time 2024-03-15T00:00:00Z \
  --end-time 2024-03-15T01:00:00Z \
  --period-in-seconds 60

# Key wait events to watch:
# db.lock.Relation           -- table lock waits
# db.io.DataFileRead         -- too many buffer misses (increase shared_buffers)
# db.CPU                     -- CPU saturation
# db.Locks.relation          -- heavy locking
```

### pg_stat_statements — Query-Level Profiling

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries by total time
SELECT
    round(total_exec_time::numeric, 2) AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS pct,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top queries by I/O
SELECT query, shared_blks_hit, shared_blks_read,
    round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_pct
FROM pg_stat_statements
ORDER BY shared_blks_read DESC LIMIT 10;

-- Reset stats
SELECT pg_stat_statements_reset();
```

---

## 13. Multi-Tenant Patterns

| Pattern | Isolation | Performance | Cost | Complexity |
|---------|-----------|-------------|------|------------|
| **Separate DB per tenant** | Full (DB-level) | Best (no cross-tenant contention) | High (one RDS per tenant) | High |
| **Separate Schema per tenant** | Schema-level | Good | Medium (one RDS, many schemas) | Medium |
| **Shared schema + tenant_id column + RLS** | Row-level (enforced at DB) | Shared resources | Low | Low |
| **Separate K8s namespace + StatefulSet** | Full (infra-level) | Best | High | Very High |

### Shared Schema with RLS (Recommended for SaaS)

```sql
-- Application sets tenant context at start of each connection
-- (done by middleware / connection pool)
SET app.tenant_id = 'tenant-uuid-here';

-- All queries automatically filtered by RLS
SELECT * FROM orders;  -- returns only current tenant's rows
INSERT INTO orders (customer_id, amount) VALUES (1, 99);  -- auto-tagged with tenant_id

-- DBA can bypass RLS (BYPASSRLS role)
SET app.tenant_id = '';  -- superuser bypass
```

---

## 14. Enterprise Database Comparison

| Feature | PostgreSQL | MySQL/Aurora MySQL | Oracle | SQL Server |
|---------|-----------|-------------------|--------|-----------|
| ACID transactions | Full | Full (InnoDB) | Full | Full |
| JSON support | JSONB (indexed, operators) | JSON (limited operators) | JSON (12c+) | JSON (2016+) |
| Partitioning | Range/List/Hash, declarative | Range/List/Hash, manually | Range/List/Hash/Composite | Range/List (partitioned tables) |
| Materialized views | Yes (manual refresh) | No (use summary tables) | Yes (automatic refresh) | Indexed views |
| Window functions | Full SQL:2003 | Full (8.0+) | Full | Full |
| Full-text search | Built-in tsvector | FULLTEXT index | Oracle Text | Full-Text Search |
| Logical replication | Yes (pglogical, built-in) | Yes (binlog, GTID) | GoldenGate, LogMiner | Transactional replication |
| Row-level security | Yes (native) | No (app-level only) | VPD (Virtual Private DB) | Row-Level Security (2016+) |
| Extensions | 1000+ (PostGIS, pg_vector, TimescaleDB) | Plugins (limited) | Oracle Extensions | CLR assemblies |
| AWS service | RDS PostgreSQL, Aurora PostgreSQL | RDS MySQL, Aurora MySQL | RDS Oracle (BYOL) | RDS SQL Server |
| License | Open source | Open source (GPL) | Commercial | Commercial |
| Recommended for | General purpose, PostGIS, analytics, complex queries | Read-heavy web apps, WordPress, existing MySQL | Oracle-dependent apps only | .NET Windows ecosystem |

---

## Quick Reference — Daily Commands

```bash
# Connect to RDS
psql -h mydb.rds.amazonaws.com -U myuser -d myapp
mysql -h mydb.rds.amazonaws.com -u myuser -p myapp

# Check RDS status
aws rds describe-db-instances --db-instance-identifier mydb \
  --query 'DBInstances[0].{Status:DBInstanceStatus,Class:DBInstanceClass,Engine:Engine}'

# RDS events (errors, restarts, maintenance)
aws rds describe-events \
  --source-identifier mydb \
  --source-type db-instance \
  --duration 1440  # last 24 hours

# Modify instance class (schedule for maintenance window or apply immediately)
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.r6g.2xlarge \
  --apply-immediately

# Enable enhanced monitoring
aws rds modify-db-instance \
  --db-instance-identifier mydb \
  --monitoring-interval 5 \            # 5-second granularity
  --monitoring-role-arn arn:aws:iam::ACCOUNT:role/rds-monitoring-role

# Force failover (Multi-AZ)
aws rds reboot-db-instance \
  --db-instance-identifier mydb \
  --force-failover
```
