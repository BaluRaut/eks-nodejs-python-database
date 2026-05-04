# Database Consolidation: RDS vs Container Postgres on EKS

This guide covers the decision between Amazon RDS and container-based PostgreSQL for a consolidation scenario: 5 separate PostgreSQL databases (approximately 20 tables each, 100 tables total) being merged into a single instance, running alongside workloads on AWS EKS.

---

## Table of Contents

1. [RDS vs Container Postgres: Decision Guide](#1-rds-vs-container-postgres-decision-guide)
2. [Merging 5 Databases into One PostgreSQL Instance](#2-merging-5-databases-into-one-postgresql-instance)
3. [RDS Setup for EKS](#3-rds-setup-for-eks)
4. [Sizing Guidance](#4-sizing-guidance)
5. [Migration Plan](#5-migration-plan)

---

## 1. RDS vs Container Postgres: Decision Guide

### RDS PostgreSQL

**Pros**

- **Managed backups**: Automated daily snapshots plus continuous WAL archiving. Point-in-time recovery (PITR) to any second within your retention window (up to 35 days) with no manual intervention.
- **Multi-AZ**: Synchronous standby replica in a second Availability Zone. Automatic failover in 60–120 seconds with no application-level changes required.
- **Automated minor version upgrades**: Security patches applied during your maintenance window without manual work.
- **IAM database authentication**: Authenticate using IAM roles and short-lived tokens instead of static passwords. Integrates cleanly with EKS Pod Identity and IRSA (IAM Roles for Service Accounts).
- **Performance Insights**: Visual query profiling, wait event analysis, and top-SQL identification built in at no extra cost for most instance sizes.
- **No operational overhead**: AWS handles OS patching, disk provisioning, replication, and crash recovery. Your team focuses on schema and queries, not infrastructure.
- **AWS-native integrations**: Enhanced Monitoring (OS-level metrics), CloudWatch Logs export, AWS Backup, and EventBridge failover notifications work out of the box.

**Cons**

- **Cost**: An `db.t3.medium` Multi-AZ instance runs roughly $80–120/month. Larger production instances (`db.r6g.large` and above) are significantly more. You pay for standby capacity even when idle.
- **Less control**: You cannot install arbitrary extensions that require superuser, modify `postgresql.conf` parameters outside the Parameter Group API, or access the underlying OS.
- **No cold-start latency concern for EKS**: Unlike Lambda-triggered RDS Proxy scenarios, EKS pods maintain persistent connections so this is not a practical issue here.

---

### Container Postgres on EKS

**Pros**

- **Cost savings**: Running `postgres:15` as a Deployment (as in `k8s/postgres.yaml` in this repo) adds no licensing cost beyond the EC2 node it runs on. If the node is already paying for your apps, the marginal cost is near zero.
- **Full control**: Any `postgresql.conf` parameter, any extension (`pg_partman`, `timescaledb`, etc.), any `pg_hba.conf` rule.
- **Co-location**: Database pod on the same node as app pods eliminates cross-AZ network latency for intra-cluster traffic.
- **Portable**: The same `postgres.yaml` manifest works in any Kubernetes cluster, local or cloud.

**Cons**

- **You own backups**: No automated snapshots. You must run `pg_dump` CronJobs, configure WAL archiving to S3 with tools like pgBackRest or WAL-G, and test restores. The `k8s/postgres.yaml` in this repo has no backup strategy at all.
- **You own upgrades**: Major version upgrades (e.g., 15 to 16) require a controlled `pg_upgrade` or dump/restore process that you plan and execute.
- **You own HA**: A single-replica Deployment (as in this repo) is a single point of failure. Setting up Patroni or Zalando Postgres Operator for automatic failover is a significant operational project.
- **PVC management**: Persistent Volume Claims on EKS are backed by EBS volumes. EBS volumes are AZ-pinned. If the node fails and the replacement is scheduled in a different AZ, the pod cannot attach the volume and your database is down until you intervene.
- **Risk of data loss**: A pod eviction or node failure without proper PVC configuration and backup procedures can result in permanent data loss. The current `postgres.yaml` in this repo uses no PVC at all — data lives in the container ephemeral layer and is destroyed on pod restart.

---

### Recommendation for This Scenario

**Use RDS PostgreSQL.**

The specific context here — 5 databases being consolidated into 1, 100 tables total, running on EKS — is exactly the workload profile where RDS pays for itself:

1. **Consolidation is a high-risk operation.** Merging schemas from 5 sources requires multiple migration attempts, data validation runs, and the ability to roll back. RDS PITR means you can restore to the second before any bad migration step without any pre-planned snapshot.

2. **One database instance is now a single point of failure for everything.** Before consolidation, a problem in one database affected one application. After consolidation, the single instance affects all five. Multi-AZ is not optional at that point; it is required for any production workload.

3. **The operational burden of container Postgres scales with the number of databases.** Managing backups, upgrades, and HA for one database is a project. Managing it correctly and reliably, while also running application workloads on the same cluster, is ongoing toil that compounds over time.

4. **100 tables is not a small schema.** At this scale, Performance Insights and Enhanced Monitoring provide query-level visibility that is difficult to replicate with self-managed Postgres on Kubernetes.

5. **IAM auth + Secrets Manager eliminates the plaintext password problem.** The current `k8s/postgres.yaml` and `k8s/node-app.yaml` in this repo store `password` in plaintext. RDS with IAM auth and AWS Secrets Manager solves this without custom tooling.

Container Postgres on EKS is appropriate for: local development, CI test environments, or stateless read replicas where losing data is acceptable.

---

## 2. Merging 5 Databases into One PostgreSQL Instance

### Strategy Options

#### Option A: One RDS instance, 5 separate PostgreSQL databases

PostgreSQL supports multiple databases per server instance. Each old database becomes a named database on the same RDS instance.

```
RDS instance: myapp-prod
  database: service_alpha    (was DB 1)
  database: service_beta     (was DB 2)
  database: service_gamma    (was DB 3)
  database: service_delta    (was DB 4)
  database: service_epsilon  (was DB 5)
```

**Pros**: Clean isolation. Each service keeps its own connection string database name. No schema conflicts possible.

**Cons**: Cross-database queries are not possible in PostgreSQL without `dblink` or `postgres_fdw`. Each database has its own set of roles and permissions to manage.

#### Option B: One database, 5 schemas (Recommended)

All five old databases become schemas inside a single PostgreSQL database. This is the recommended approach for consolidation.

```
RDS instance: myapp-prod
  database: myapp
    schema: alpha    (was DB 1)
    schema: beta     (was DB 2)
    schema: gamma    (was DB 3)
    schema: delta    (was DB 4)
    schema: epsilon  (was DB 5)
```

**Pros**: Cross-schema queries work natively with no extensions. Single connection string database name. Unified role management. Easier to enforce consistent policies.

**Cons**: Requires updating all application queries that assume `public` schema or unqualified table names.

#### Option C: Flat merge into public schema

All 100 tables in a single `public` schema with naming prefixes (`alpha_users`, `beta_users`, etc.).

**Avoid this approach.** It loses all logical separation, creates naming convention debt, and makes it harder to grant fine-grained permissions per service.

---

### Recommended Approach: Schema-Based Isolation

#### Create schemas

```sql
-- Connect to the target database
\c myapp

CREATE SCHEMA alpha;
CREATE SCHEMA beta;
CREATE SCHEMA gamma;
CREATE SCHEMA delta;
CREATE SCHEMA epsilon;
```

#### Create per-service roles with schema-scoped access

```sql
-- Role for the alpha service
CREATE ROLE alpha_app LOGIN PASSWORD 'use-iam-auth-instead';
GRANT CONNECT ON DATABASE myapp TO alpha_app;
GRANT USAGE ON SCHEMA alpha TO alpha_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA alpha TO alpha_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA alpha
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO alpha_app;

-- Repeat for each service role
CREATE ROLE beta_app LOGIN PASSWORD 'use-iam-auth-instead';
GRANT CONNECT ON DATABASE myapp TO beta_app;
GRANT USAGE ON SCHEMA beta TO beta_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA beta TO beta_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA beta
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO beta_app;
```

#### Set search_path per role so applications need no query changes

```sql
ALTER ROLE alpha_app SET search_path TO alpha, public;
ALTER ROLE beta_app  SET search_path TO beta,  public;
ALTER ROLE gamma_app SET search_path TO gamma, public;
ALTER ROLE delta_app SET search_path TO delta, public;
ALTER ROLE epsilon_app SET search_path TO epsilon, public;
```

With `search_path` set, an application that previously ran `SELECT * FROM users` on DB 1 now runs the same query on the consolidated database as `alpha_app` and PostgreSQL resolves it as `alpha.users`. No application query changes are needed.

#### Cross-schema query example

One advantage of schema consolidation over separate databases is native cross-schema joins:

```sql
-- Join a table from the alpha service with a table from the beta service
SELECT
    a.id          AS alpha_user_id,
    a.email,
    b.order_id,
    b.total
FROM alpha.users a
JOIN beta.orders b ON a.id = b.customer_id
WHERE b.created_at > NOW() - INTERVAL '30 days';
```

This is not possible across separate PostgreSQL databases without `postgres_fdw`.

---

### Cross-Database Query Limitation

PostgreSQL does not allow a query to reference tables in a different database in the same statement. This is a fundamental architectural constraint, not a configuration option.

**If you choose Option A (5 separate databases)**, you must use `postgres_fdw` for any cross-service queries:

```sql
-- On the alpha database, create a foreign server pointing to the beta database
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER beta_server
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'myapp-prod.cluster-xxxx.us-east-1.rds.amazonaws.com', dbname 'beta', port '5432');

CREATE USER MAPPING FOR alpha_app
  SERVER beta_server
  OPTIONS (user 'beta_app', password '...');

-- Import the remote schema
IMPORT FOREIGN SCHEMA public
  FROM SERVER beta_server
  INTO beta_foreign;

-- Now you can query
SELECT * FROM beta_foreign.orders WHERE customer_id = 42;
```

`postgres_fdw` works but adds latency, complexity, and a dependency between databases that makes schema consolidation (Option B) preferable in almost all cases.

---

## 3. RDS Setup for EKS

### VPC and Network Configuration

RDS must be in the same VPC as your EKS cluster. EKS nodes communicate with RDS over private subnets; the RDS instance should never have a public endpoint.

#### Create a DB subnet group spanning private subnets

```bash
# Get your private subnet IDs (the ones your EKS nodes use)
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-xxxxxxxxx" "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
  --query 'Subnets[*].SubnetId' \
  --output text

# Create the subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name myapp-rds-subnet-group \
  --db-subnet-group-description "RDS subnet group for EKS workloads" \
  --subnet-ids subnet-aaa subnet-bbb subnet-ccc
```

#### Create a security group for RDS

```bash
# Create the security group
aws ec2 create-security-group \
  --group-name myapp-rds-sg \
  --description "Allow Postgres from EKS nodes" \
  --vpc-id vpc-xxxxxxxxx

# Allow port 5432 from the EKS node security group only
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds-xxxxxxxxx \
  --protocol tcp \
  --port 5432 \
  --source-group sg-eks-nodes-xxxxxxxxx
```

Do not open port 5432 to `0.0.0.0/0`. Only the EKS node security group (or specific pod CIDR if using security groups for pods) should have access.

#### Create the RDS instance

```bash
aws rds create-db-instance \
  --db-instance-identifier myapp-prod \
  --db-instance-class db.r6g.large \
  --engine postgres \
  --engine-version 15.6 \
  --master-username myapp_admin \
  --master-user-password "$(aws secretsmanager get-secret-value --secret-id myapp/rds/master --query SecretString --output text | jq -r .password)" \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --multi-az \
  --db-subnet-group-name myapp-rds-subnet-group \
  --vpc-security-group-ids sg-rds-xxxxxxxxx \
  --backup-retention-period 14 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --enable-performance-insights \
  --no-publicly-accessible \
  --deletion-protection
```

---

### Storing Credentials: Secrets Manager + Kubernetes

Never store database passwords in plaintext `env` values in your pod specs (as the current `k8s/node-app.yaml` does). Use AWS Secrets Manager with the Kubernetes Secrets Store CSI Driver, or sync to a Kubernetes Secret.

#### Option A: Kubernetes Secret (simpler, acceptable for most teams)

```bash
# Store the connection string in Secrets Manager
aws secretsmanager create-secret \
  --name myapp/rds/alpha-service \
  --secret-string '{
    "host": "myapp-prod.cluster-xxxx.us-east-1.rds.amazonaws.com",
    "port": "5432",
    "dbname": "myapp",
    "username": "alpha_app",
    "password": "your-strong-password"
  }'

# Create a Kubernetes Secret from it (run this in your CI/CD pipeline or setup script)
kubectl create secret generic alpha-db-secret \
  --from-literal=DB_HOST="myapp-prod.cluster-xxxx.us-east-1.rds.amazonaws.com" \
  --from-literal=DB_PORT="5432" \
  --from-literal=DB_NAME="myapp" \
  --from-literal=DB_USER="alpha_app" \
  --from-literal=DB_PASSWORD="your-strong-password"
```

Reference the secret in your pod spec:

```yaml
# k8s/node-app.yaml (updated for RDS)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
      - name: node-app
        image: 425848620191.dkr.ecr.us-east-1.amazonaws.com/node-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: alpha-db-secret
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            secretKeyRef:
              name: alpha-db-secret
              key: DB_PORT
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: alpha-db-secret
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: alpha-db-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: alpha-db-secret
              key: DB_PASSWORD
```

#### Option B: Secrets Store CSI Driver (recommended for production)

The CSI driver syncs secrets from AWS Secrets Manager directly into pod filesystems or environment variables without you manually copying values into Kubernetes Secrets.

```bash
# Install the CSI driver and AWS provider
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true

helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm install secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws \
  --namespace kube-system
```

```yaml
# SecretProviderClass to mount Secrets Manager values
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: alpha-db-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "myapp/rds/alpha-service"
        objectType: "secretsmanager"
        jmesPath:
          - path: host
            objectAlias: DB_HOST
          - path: password
            objectAlias: DB_PASSWORD
  secretObjects:
  - secretName: alpha-db-secret
    type: Opaque
    data:
    - objectName: DB_HOST
      key: DB_HOST
    - objectName: DB_PASSWORD
      key: DB_PASSWORD
```

---

### RDS Proxy for Connection Pooling

EKS workloads open many short-lived connections — each pod replica maintains its own pool, and if you have 5 services with 3 replicas each, that is 15+ connection pools hitting the same RDS instance. PostgreSQL forks a backend process per connection. On smaller instance classes this exhausts memory quickly.

RDS Proxy sits between your pods and RDS, multiplexing thousands of application connections into a small, stable set of database connections.

```bash
# Create an RDS Proxy
aws rds create-db-proxy \
  --db-proxy-name myapp-proxy \
  --engine-family POSTGRESQL \
  --auth '[{
    "AuthScheme": "SECRETS",
    "SecretArn": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/rds/alpha-service",
    "IAMAuth": "REQUIRED"
  }]' \
  --role-arn arn:aws:iam::123456789:role/rds-proxy-role \
  --vpc-subnet-ids subnet-aaa subnet-bbb subnet-ccc \
  --vpc-security-group-ids sg-rds-xxxxxxxxx

# Register the RDS instance as the proxy target
aws rds register-db-proxy-targets \
  --db-proxy-name myapp-proxy \
  --db-instance-identifiers myapp-prod
```

Update your application's `DB_HOST` to point to the proxy endpoint instead of the RDS endpoint:

```
myapp-proxy.proxy-xxxx.us-east-1.rds.amazonaws.com
```

The proxy endpoint is available in the RDS console under "Proxies" or via:

```bash
aws rds describe-db-proxies \
  --db-proxy-name myapp-proxy \
  --query 'DBProxies[0].Endpoint'
```

**When to use RDS Proxy**: If your total pod count across all services is more than 20, or if you anticipate scaling beyond that, enable RDS Proxy from the start. It is easier to deploy before a connection exhaustion incident than during one.

---

## 4. Sizing Guidance

### Instance Class

100 tables with approximately 20 tables per service is a moderate schema size. The right instance class depends on data volume and query concurrency, not table count alone. Use these as starting points:

| Scenario | Instance Class | vCPU | RAM | Notes |
|---|---|---|---|---|
| Development / staging | db.t3.medium | 2 | 4 GB | Burstable. Do not use for production. |
| Production, low traffic (< 100 concurrent connections) | db.t3.large | 2 | 8 GB | Burstable. Acceptable for internal tools or low-traffic APIs. |
| Production, moderate traffic | db.r6g.large | 2 | 16 GB | Graviton2, memory-optimized. Recommended starting point. |
| Production, high traffic or analytics queries | db.r6g.xlarge | 4 | 32 GB | Step up if query times degrade under load. |

For a fresh consolidation of 5 databases, start with `db.r6g.large` Multi-AZ. Monitor CPU and FreeableMemory in CloudWatch for the first two weeks after migration and scale up if needed. Scaling an RDS instance vertically requires a reboot (a few minutes of downtime unless Multi-AZ, in which case it fails over automatically).

### Storage

**Use gp3.** It is strictly better than gp2 for most workloads: 20% cheaper, and IOPS/throughput are configurable independently of storage size.

```bash
# gp3 defaults: 3000 IOPS and 125 MB/s throughput included at no extra cost
# You can provision up to 16000 IOPS and 1000 MB/s on gp3

# For 100 tables starting out, 100 GB is more than enough for schema + indexes
# Enable autoscaling so you never need to manually expand
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --storage-type gp3 \
  --allocated-storage 100 \
  --max-allocated-storage 500   # autoscales up to 500 GB automatically
```

**When to use io1/io2**: Only if your workload requires sustained IOPS above 16,000 (the gp3 ceiling) or you need sub-millisecond consistent latency for write-heavy OLTP at scale. This is unlikely for a 100-table consolidated database unless you are processing millions of writes per minute.

### Multi-AZ

Enable Multi-AZ for any database that serves production traffic. The cost is approximately 2x the single-AZ price (you pay for the standby instance), but the standby replica:

- Serves as an automatic failover target (60–120 second RTO, zero RPO for committed transactions)
- Can be promoted manually for maintenance
- Does not add read capacity (for read scaling, use Read Replicas separately)

```bash
# Enable Multi-AZ on an existing instance during a maintenance window
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --multi-az \
  --apply-immediately   # omit this to apply during next maintenance window
```

---

## 5. Migration Plan

### Overview

The migration has four phases:

1. Set up the RDS target and create schemas
2. Dump source databases and restore into target schemas
3. Validate data and run applications in parallel
4. Cut over connection strings and decommission old databases

### Phase 1: Prepare the RDS target

```sql
-- Connect to the new RDS instance as master user
CREATE DATABASE myapp;
\c myapp

CREATE SCHEMA alpha;
CREATE SCHEMA beta;
CREATE SCHEMA gamma;
CREATE SCHEMA delta;
CREATE SCHEMA epsilon;

-- Create application roles (see section 2 for full GRANT statements)
CREATE ROLE alpha_app LOGIN;
-- ... repeat for each service
```

### Phase 2: Dump and restore with pg_dump / pg_restore

Dump each source database into its target schema. The `--schema` flag on `pg_restore` and `--no-owner` / `--no-acl` flags prevent permission conflicts.

```bash
# For each of the 5 source databases, repeat this pattern

# Step 1: Dump the source database (run on the source host or a machine with access)
pg_dump \
  --host=source-db-1.example.com \
  --port=5432 \
  --username=postgres \
  --dbname=service_alpha_db \
  --format=custom \
  --no-owner \
  --no-acl \
  --file=/tmp/alpha_dump.pgdump

# Step 2: Restore into the target schema on RDS
pg_restore \
  --host=myapp-prod.cluster-xxxx.us-east-1.rds.amazonaws.com \
  --port=5432 \
  --username=myapp_admin \
  --dbname=myapp \
  --schema=alpha \         # restore all objects into the alpha schema
  --no-owner \
  --no-acl \
  --jobs=4 \               # parallel restore jobs, speeds up large dumps
  /tmp/alpha_dump.pgdump
```

If the source databases use the `public` schema (the default), pg_dump will export objects into `public`. You can remap them to the target schema using `--schema=public` on the dump side and a sed pass on the SQL, or use the `--schema` option on restore to redirect to the correct target schema. For a cleaner approach:

```bash
# Dump as plain SQL, then sed-replace the schema name before restoring
pg_dump \
  --host=source-db-1.example.com \
  --username=postgres \
  --dbname=service_alpha_db \
  --format=plain \
  --no-owner \
  --no-acl \
  --schema=public \
  > /tmp/alpha.sql

# Replace "public." references with "alpha." in the SQL file
sed -i 's/public\./alpha\./g; s/SET search_path = public/SET search_path = alpha/g' /tmp/alpha.sql

# Restore
psql \
  --host=myapp-prod.cluster-xxxx.us-east-1.rds.amazonaws.com \
  --username=myapp_admin \
  --dbname=myapp \
  --file=/tmp/alpha.sql
```

### Phase 3: Data validation

Before cutting over, verify row counts match between source and target.

```sql
-- On the source database
SELECT
    schemaname,
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY tablename;

-- On the target RDS instance (for the alpha schema)
SELECT
    schemaname,
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
WHERE schemaname = 'alpha'
ORDER BY tablename;
```

Also verify checksums on critical tables:

```sql
-- Run on both source and target for each important table
SELECT COUNT(*), SUM(hashtext(t::text)) AS checksum
FROM alpha.users t;
```

### Phase 4: Cut over connection strings

Update each application's database connection string one service at a time. With schema-based isolation and `search_path` set per role, the only change required is:

| Parameter | Old value | New value |
|---|---|---|
| `DB_HOST` | `postgres-service` (in-cluster) or old host | RDS Proxy endpoint or RDS endpoint |
| `DB_NAME` | `service_alpha_db` | `myapp` |
| `DB_USER` | `postgres` | `alpha_app` |
| `DB_PASSWORD` | old password | new password from Secrets Manager |

Update the Kubernetes Secret and trigger a rolling restart:

```bash
# Update the secret
kubectl create secret generic alpha-db-secret \
  --from-literal=DB_HOST="myapp-proxy.proxy-xxxx.us-east-1.rds.amazonaws.com" \
  --from-literal=DB_NAME="myapp" \
  --from-literal=DB_USER="alpha_app" \
  --from-literal=DB_PASSWORD="$(aws secretsmanager get-secret-value \
      --secret-id myapp/rds/alpha-service \
      --query SecretString --output text | jq -r .password)" \
  --dry-run=client -o yaml | kubectl apply -f -

# Rolling restart to pick up the new secret values
kubectl rollout restart deployment/node-app
kubectl rollout status deployment/node-app
```

---

### Live Migration with AWS DMS (minimal downtime)

For production databases that cannot tolerate a maintenance window for the dump/restore, use AWS Database Migration Service to replicate changes continuously while the migration runs.

**DMS migration phases:**

1. Full load: DMS copies all existing rows from source to target.
2. Change Data Capture (CDC): DMS reads the source database's WAL (Write-Ahead Log) and applies ongoing changes to the target in near real time.
3. Cutover: When the target is fully caught up (replication lag near zero), you update connection strings and stop DMS.

```bash
# Create a replication instance
aws dms create-replication-instance \
  --replication-instance-identifier myapp-dms \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 50 \
  --vpc-security-group-ids sg-rds-xxxxxxxxx \
  --publicly-accessible false

# Create source endpoint (one per source database)
aws dms create-endpoint \
  --endpoint-identifier source-alpha \
  --endpoint-type source \
  --engine-name postgres \
  --server-name source-db-1.example.com \
  --port 5432 \
  --database-name service_alpha_db \
  --username postgres \
  --password "$SOURCE_PASSWORD"

# Create target endpoint (the RDS instance)
aws dms create-endpoint \
  --endpoint-identifier target-rds \
  --endpoint-type target \
  --engine-name postgres \
  --server-name myapp-prod.cluster-xxxx.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --database-name myapp \
  --username myapp_admin \
  --password "$MASTER_PASSWORD"

# Create and start the replication task
aws dms create-replication-task \
  --replication-task-identifier migrate-alpha \
  --source-endpoint-arn arn:aws:dms:...:endpoint:source-alpha \
  --target-endpoint-arn arn:aws:dms:...:endpoint:target-rds \
  --replication-instance-arn arn:aws:dms:...:rep:myapp-dms \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [{
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all",
      "object-locator": {"schema-name": "public", "table-name": "%"},
      "rule-action": "include"
    }, {
      "rule-type": "transformation",
      "rule-id": "2",
      "rule-name": "rename-schema",
      "rule-action": "convert-schema",
      "rule-target": "schema",
      "object-locator": {"schema-name": "public"},
      "value": "alpha"
    }]
  }'

aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:...:task:migrate-alpha \
  --start-replication-task-type start-replication

# Monitor replication lag
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:...:task:migrate-alpha \
  --query 'ReplicationTasks[0].ReplicationTaskStats'
```

**Source database prerequisites for CDC**: The source PostgreSQL instance must have `wal_level = logical`. For RDS sources, enable this in the Parameter Group. For self-managed Postgres, set it in `postgresql.conf` and restart.

```sql
-- Verify on the source
SHOW wal_level;
-- Must return: logical
```

---

### Removing the In-Cluster Container Database

Once all services are validated against RDS, remove the container Postgres from EKS:

```bash
kubectl delete deployment postgres
kubectl delete service postgres-service

# If you had a PVC
kubectl delete pvc postgres-pvc

# Optionally delete the postgres.yaml manifest or update it to document
# that it is no longer used for production
```

The `k8s/postgres.yaml` in this repository is useful for local development and testing. Keep it available for that purpose but do not use it for persistent production data.
