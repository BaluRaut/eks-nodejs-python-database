# Database Migration & DR Sequence Diagrams

Sequence diagrams for every major database migration path, disaster recovery scenario, backup/restore flow, and K8s database operation.

---

## 1. On-Prem PostgreSQL to Aurora PostgreSQL — Online Migration (Zero Downtime)

```mermaid
sequenceDiagram
    participant OnPrem as On-Prem PostgreSQL
    participant DMS as AWS DMS Replication Instance
    participant Aurora as Aurora PostgreSQL (target)
    participant App as Application
    participant DNS as Connection String / DNS

    note over OnPrem,Aurora: Phase 1 - Initial full load (app still points to on-prem)

    DMS->>OnPrem: Connect to source (pg_logical or wal2json)
    DMS->>Aurora: Create target schema (via AWS SCT output)
    DMS->>OnPrem: Full load - SELECT all rows in batches
    OnPrem-->>DMS: Stream all existing rows
    DMS->>Aurora: Bulk INSERT rows to Aurora
    note over DMS: Full load takes hours to days depending on data volume

    note over OnPrem,Aurora: Phase 2 - CDC (Change Data Capture) catches up

    DMS->>OnPrem: Read WAL stream from replication slot
    OnPrem-->>DMS: INSERT UPDATE DELETE changes since full load started
    DMS->>Aurora: Apply changes to Aurora in near real-time
    note over DMS: Replication lag monitored - target: under 1 second

    note over OnPrem,Aurora: Phase 3 - Cutover (5-minute planned maintenance window)

    App->>App: Maintenance mode ON - reject new writes
    DMS->>DMS: Wait for lag to reach 0 seconds
    DMS-->>App: Signal cutover ready
    App->>DNS: Update DB connection string to Aurora endpoint
    App->>App: Maintenance mode OFF
    App->>Aurora: New writes go to Aurora
    note over OnPrem,Aurora: On-prem kept running for 48h rollback window then decommissioned
```

---

## 2. Oracle to Aurora PostgreSQL — Schema Conversion + DMS

```mermaid
sequenceDiagram
    participant Oracle as Oracle DB (source)
    participant SCT as AWS Schema Conversion Tool
    participant Aurora as Aurora PostgreSQL (target)
    participant DMS as AWS DMS
    participant Dev as Developer (manual fixes)

    note over Oracle,Dev: Phase 1 - Schema Assessment and Conversion

    SCT->>Oracle: Connect and scan all objects
    Oracle-->>SCT: Tables views procs triggers sequences packages
    SCT->>SCT: Analyze conversion complexity
    SCT-->>Dev: Assessment report - auto-converted vs manual-fix items
    note over SCT: Typical Oracle->PG: 60-80 percent auto-converted

    Dev->>SCT: Review and fix PL/SQL to PL/pgSQL incompatibilities
    note over Dev: Common fixes: ROWNUM to LIMIT, SYSDATE to now(), sequences, DECODE to CASE

    SCT->>Aurora: Generate and apply converted DDL scripts
    Aurora-->>SCT: Schema created

    note over Oracle,Aurora: Phase 2 - Data Migration via DMS

    DMS->>Oracle: Full load using SELECT on all tables
    Oracle-->>DMS: Row data
    DMS->>Aurora: Insert rows with type mapping (NUMBER to NUMERIC, DATE to TIMESTAMP)

    note over Oracle,Aurora: Phase 3 - CDC using Oracle LogMiner

    DMS->>Oracle: Enable supplemental logging
    DMS->>Oracle: Stream redo logs via LogMiner
    Oracle-->>DMS: DML changes INSERT UPDATE DELETE
    DMS->>Aurora: Apply changes
    DMS-->>Dev: Lag metrics and error counts

    note over Oracle,Aurora: Phase 4 - Validation and cutover

    Dev->>Oracle: Row count and checksum validation
    Dev->>Aurora: Row count and checksum validation
    Dev->>Dev: Compare results - resolve discrepancies
    Dev->>App: Update JDBC/connection string to Aurora
    Dev->>Oracle: Archive and decommission Oracle license
```

---

## 3. MySQL to Aurora MySQL — Near-Zero Downtime via Snapshot

```mermaid
sequenceDiagram
    participant RDS as RDS MySQL (source)
    participant Snap as RDS Snapshot
    participant Aurora as Aurora MySQL (target)
    participant App as Application
    participant DMS as AWS DMS (CDC only)

    note over RDS,Aurora: Homogeneous migration - simplest path

    note over RDS,Snap: Step 1 - Take snapshot of RDS MySQL

    App->>App: Continue serving traffic on RDS MySQL
    RDS->>Snap: Create manual snapshot (no downtime)
    Snap-->>RDS: Snapshot complete - captures point-in-time

    note over Snap,Aurora: Step 2 - Restore snapshot to Aurora MySQL

    Snap->>Aurora: Restore to Aurora MySQL cluster
    note over Aurora: Takes 30-60 min for large databases

    note over Aurora,DMS: Step 3 - Set up CDC to replay changes made during snapshot restore

    DMS->>RDS: Enable binary log (binlog_format=ROW already required)
    DMS->>Aurora: Apply changes since snapshot was taken
    DMS-->>App: Signal lag = 0 ready for cutover

    note over App,Aurora: Step 4 - Cutover (seconds of downtime)

    App->>App: Drain connections - no new transactions
    App->>App: Wait for in-flight transactions to complete
    App->>Aurora: Update connection string to Aurora cluster endpoint
    App-->>App: Resume traffic on Aurora MySQL
    note over Aurora: Aurora now serves all traffic - RDS MySQL kept as rollback for 24h
```

---

## 4. SQL Server to Aurora PostgreSQL — Enterprise Migration

```mermaid
sequenceDiagram
    participant MSSQL as SQL Server (source)
    participant SCT as AWS SCT
    participant Aurora as Aurora PostgreSQL (target)
    participant SSIS as SSIS or DMS
    participant Dev as Migration Team

    note over MSSQL,Dev: Phase 1 - Schema and code conversion

    SCT->>MSSQL: Scan all schemas objects stored procs views
    MSSQL-->>SCT: T-SQL objects and schema
    SCT-->>Dev: Report - estimated effort by object type

    Dev->>Dev: Convert T-SQL to PL/pgSQL manually for complex procs
    note over Dev: T-SQL differences: TOP vs LIMIT, GETDATE vs now(), ISNULL vs COALESCE

    Dev->>SCT: Apply converted schema to Aurora
    SCT->>Aurora: CREATE TABLE CREATE INDEX sequences
    Aurora-->>Dev: Schema validated

    note over MSSQL,Aurora: Phase 2 - Data migration

    SSIS->>MSSQL: Read tables via SQL Server bulk copy
    MSSQL-->>SSIS: Row batches
    SSIS->>Aurora: Write via COPY or batch INSERT
    note over SSIS: Use DMS for ongoing CDC after initial load

    note over MSSQL,Aurora: Phase 3 - Application changes

    Dev->>Dev: Update connection strings from JDBC:sqlserver to JDBC:postgresql
    Dev->>Dev: Replace SQL Server driver with PostgreSQL JDBC
    Dev->>Dev: Fix SQL dialect issues in application code
    Dev->>Dev: Test all stored procedures and queries
    Dev->>Aurora: Run regression test suite
    Aurora-->>Dev: All tests pass

    note over Dev,Aurora: Phase 4 - Cutover and validation

    Dev->>MSSQL: Final consistency check row counts and checksums
    Dev->>Aurora: Validate row counts match
    Dev->>App: Deploy app with Aurora connection strings
    Dev->>MSSQL: Keep in read-only mode for 1 week as rollback option
```

---

## 5. K8s PostgreSQL StatefulSet — Backup to S3 with CronJob

```mermaid
sequenceDiagram
    participant Cron as K8s CronJob (scheduler)
    participant Job as Backup Job Pod
    participant PG as PostgreSQL StatefulSet
    participant S3 as AWS S3 Bucket
    participant SNS as SNS Alert Topic

    note over Cron,SNS: Daily backup at 1 AM

    Cron->>Job: Create Job pod at 01:00 UTC
    Job->>Job: Pod starts - load env vars from Secret

    Job->>PG: pg_dump -h postgres-headless -Fc -f /tmp/backup.dump myapp
    PG-->>Job: Streaming dump output
    Job->>Job: Verify dump file size is nonzero

    alt Dump failed or zero bytes
        Job->>SNS: Publish backup-failed alert with pod logs
        SNS-->>Dev: PagerDuty / email alert
        Job->>Job: Exit code 1 - Job marked Failed
    end

    Job->>S3: aws s3 cp /tmp/backup.dump s3://backups/postgres/YYYY-MM-DD/backup.dump
    S3-->>Job: Upload complete with ETag

    Job->>S3: List objects older than 30 days and delete (lifecycle or manual)
    Job->>SNS: Publish backup-success with size and duration
    SNS-->>Dev: Slack notification backup complete 4.2GB in 12min

    note over Cron,S3: S3 bucket has versioning and cross-region replication enabled.<br/>Backups encrypted with SSE-KMS.
```

---

## 6. Aurora Multi-AZ Failover — Step-by-Step

```mermaid
sequenceDiagram
    participant App as Application (PgBouncer pool)
    participant Writer as Aurora Writer AZ-A
    participant Storage as Aurora Distributed Storage (6 copies 3 AZs)
    participant Replica as Aurora Replica AZ-B
    participant DNS as Aurora Cluster DNS Endpoint
    participant CW as CloudWatch Health Check

    note over App,CW: Normal operation - writer in AZ-A

    App->>Writer: Write queries via cluster endpoint DNS
    Writer->>Storage: Write to 4 of 6 storage nodes (quorum)
    Storage-->>Writer: ACK
    Writer-->>App: Write confirmed

    note over Writer,CW: AZ-A failure or writer crash

    Writer->>Writer: Instance becomes unavailable T=0
    CW->>CW: Health check fails 3 times (15 seconds)
    CW->>Replica: Trigger failover - promote AZ-B replica

    note over Replica,DNS: Promotion sequence (20-60 seconds)

    Replica->>Replica: Become new writer - open for writes T+15s
    DNS->>DNS: Update cluster endpoint DNS to point to AZ-B T+20s
    note over DNS: DNS TTL is 5 seconds - apps reconnect quickly

    App->>App: PgBouncer detects broken connections
    App->>App: Reconnect with retry logic
    App->>DNS: Resolve cluster endpoint
    DNS-->>App: New writer IP AZ-B
    App->>Replica: Connected to new writer T+30s

    note over App,Replica: Old writer AZ-A auto-restarts as a read replica
    Writer->>Writer: Instance restarts in AZ-A as replica

    note over App,Replica: RPO = 0 (shared storage - no replication lag).<br/>RTO = 20-60 seconds (automatic no manual intervention).
```

---

## 7. Point-in-Time Recovery (PITR)

```mermaid
sequenceDiagram
    participant Dev as DBA / Developer
    participant RDS as RDS or Aurora Instance
    participant Backup as Automated Backup System
    participant S3 as S3 (WAL / Binlog archive)
    participant NewDB as Restored DB Instance

    note over Dev,NewDB: Scenario - accidental DROP TABLE at 14:32 UTC

    Dev->>Dev: Alert - table orders missing at 14:35 UTC

    note over Dev,S3: How PITR works

    Backup->>RDS: Take automated daily snapshot at 00:00 UTC
    Backup->>S3: Continuously ship WAL (PostgreSQL) or binlogs (MySQL) to S3
    note over S3: WAL archive enables restoring to any second within retention period (35 days max)

    note over Dev,NewDB: Restore to 14:30 UTC (2 minutes before the DROP)

    Dev->>RDS: Restore to point in time - target 14:30 UTC
    RDS->>NewDB: Launch new instance from last snapshot (00:00 backup)
    NewDB->>S3: Replay WAL changes from 00:00 to 14:30 UTC
    S3-->>NewDB: Apply 14.5 hours of WAL changes
    note over NewDB: Instance ready with data as of 14:30 UTC - takes 30-60 min

    Dev->>NewDB: Dump the dropped table from restored instance
    NewDB-->>Dev: pg_dump orders table data
    Dev->>RDS: Restore orders table to production via pg_restore
    RDS-->>Dev: Table and data restored

    note over Dev,NewDB: Production continues running during restore.<br/>Only the specific table is moved back - not a full cutover.
```

---

## 8. Aurora Global Database — Planned Region Failover

```mermaid
sequenceDiagram
    participant AppUS as App (us-east-1)
    participant PrimaryUS as Aurora Primary (us-east-1)
    participant StorageUS as Storage us-east-1
    participant StorageEU as Storage eu-west-1
    participant SecondaryEU as Aurora Secondary (eu-west-1)
    participant AppEU as App (eu-west-1)
    participant Ops as Operations Team

    note over AppUS,AppEU: Planned regional maintenance - failover eu-west-1

    AppUS->>PrimaryUS: Writes flowing normally
    PrimaryUS->>StorageUS: Write to local storage
    StorageUS->>StorageEU: Async replication 1 sec lag
    AppEU->>SecondaryEU: Reads from local secondary (low latency)

    note over Ops,SecondaryEU: Initiate managed failover

    Ops->>PrimaryUS: aws rds failover-global-cluster target=eu-west-1
    PrimaryUS->>PrimaryUS: Demote to secondary
    note over PrimaryUS: Demoted primary stops accepting writes

    SecondaryEU->>SecondaryEU: Promoted to primary writer
    SecondaryEU->>StorageEU: Now accepts writes
    note over SecondaryEU: Promotion completes in under 1 minute

    Ops->>AppUS: Update connection strings to eu-west-1 cluster endpoint
    Ops->>AppEU: Update connection strings to eu-west-1 cluster endpoint (local writer now)
    AppEU->>SecondaryEU: Writes with low latency
    AppUS->>SecondaryEU: Writes cross-region (higher latency acceptable for maintenance)

    note over PrimaryUS,SecondaryEU: Old primary (us-east-1) now replicates from eu-west-1.<br/>After maintenance - reverse failover back to us-east-1 optionally.
```

---

## 9. Database Schema Migration — K8s Rolling Deploy

```mermaid
sequenceDiagram
    participant CI as CI Pipeline
    participant K8s as Kubernetes
    participant InitC as Init Container (Flyway)
    participant DB as PostgreSQL
    participant AppV1 as App v1 Pods (running)
    participant AppV2 as App v2 Pods (deploying)

    note over CI,AppV2: Step 1 - Push new app version with additive migration

    CI->>K8s: kubectl apply deployment with image tag v2
    K8s->>InitC: Start init container before any v2 app pods

    InitC->>DB: flyway migrate - run V5__add_email_column.sql
    DB->>DB: ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT false
    DB-->>InitC: Migration V5 applied
    InitC-->>K8s: Init container exit code 0 - migration success

    note over K8s,AppV1: Step 2 - Rolling update (v1 and v2 run simultaneously)

    K8s->>AppV2: Start first v2 pod
    AppV2->>DB: Reads and writes new column email_verified
    AppV1->>DB: Still writing - ignores new column (backward compatible)
    note over AppV1,AppV2: Both versions work with the same schema simultaneously

    K8s->>AppV1: Terminate first v1 pod
    K8s->>AppV2: Start second v2 pod
    K8s->>AppV1: Terminate last v1 pod

    note over K8s,AppV2: All traffic now on v2

    note over CI,AppV2: Step 3 - Optional cleanup migration (after v1 fully retired)

    CI->>K8s: Deploy v2.1 with V6__set_not_null.sql
    InitC->>DB: ALTER COLUMN email_verified SET NOT NULL
    DB-->>InitC: Constraint added (backfill already done)
```

---

## 10. Read Replica Promotion — DR Runbook

```mermaid
sequenceDiagram
    participant Ops as On-Call Engineer
    participant Primary as RDS Primary (FAILED)
    participant Replica as RDS Read Replica (cross-region)
    participant DNS as Application DNS / Secrets Manager
    participant App as Application Pods
    participant Runbook as Automated Runbook (SSM / Lambda)

    note over Ops,Runbook: Unplanned outage - primary region down

    Primary->>Primary: Region outage detected T=0
    Ops->>Ops: PagerDuty alert - DB health check failed T+2min

    note over Ops,Runbook: Manual promotion of read replica

    Ops->>Replica: aws rds promote-read-replica --db-instance-identifier mydb-replica
    Replica->>Replica: Stop replication from primary
    Replica->>Replica: Apply remaining relay logs
    Replica->>Replica: Open for writes T+5min
    Replica-->>Ops: Replica promoted - new endpoint available

    note over Ops,App: Update application to point to promoted replica

    Ops->>DNS: Update Secrets Manager DB_HOST to new replica endpoint
    note over DNS: External Secrets Operator will pick this up within 1 minute

    App->>App: ESO syncs new DB_HOST to K8s Secret
    App->>App: Pod env var updated - rolling restart triggered
    App->>Replica: Connect to promoted replica
    Replica-->>App: Connection accepted - serving reads and writes

    note over Ops,Replica: RPO = replication lag at time of failure (seconds to minutes).<br/>RTO = 5-15 minutes (manual process).<br/>Automate with Route53 health checks + Lambda for RTO under 2 minutes.
```

---

## 11. DynamoDB Global Tables — Multi-Region Write

```mermaid
sequenceDiagram
    participant AppUS as App (us-east-1)
    participant DDB_US as DynamoDB us-east-1
    participant DDB_EU as DynamoDB eu-west-1
    participant AppEU as App (eu-west-1)
    participant Conflict as Last-Write-Wins Resolver

    note over AppUS,AppEU: Global Tables - every region is a writer

    AppUS->>DDB_US: PutItem userId=u1 name=Alice region=US timestamp=T1
    DDB_US-->>AppUS: 200 Written locally T1

    AppEU->>DDB_EU: PutItem userId=u1 name=Alice EU region=EU timestamp=T2
    DDB_EU-->>AppEU: 200 Written locally T2

    note over DDB_US,DDB_EU: Async cross-region replication (under 1 second)

    DDB_US->>DDB_EU: Replicate write for userId=u1 from US
    DDB_EU->>DDB_EU: Receive US write - same userId=u1 with different timestamp

    alt Conflict: both regions wrote to same item nearly simultaneously
        DDB_EU->>Conflict: Compare timestamps T1 vs T2
        Conflict->>DDB_EU: Last write wins - T2 is newer so keep EU version
        note over Conflict: DynamoDB Global Tables use last-write-wins by timestamp.<br/>Custom conflict resolution not natively supported.
    end

    DDB_EU->>DDB_US: Replicate EU version back (if EU won)
    DDB_US->>DDB_US: Item now shows EU value in all regions

    note over AppUS,AppEU: Reads always from local region = single-digit ms latency.<br/>Writes accepted locally = no cross-region write latency.
```

---

## 12. Cross-Account Database Snapshot — Compliance Backup

```mermaid
sequenceDiagram
    participant ProdAccount as Production AWS Account
    participant RDS as RDS Instance (prod)
    participant ProdSnap as Snapshot (prod account)
    participant Backup as Backup AWS Account (isolated)
    participant BackupSnap as Snapshot Copy (backup account)
    participant Glacier as S3 Glacier (long-term)

    note over ProdAccount,Glacier: Daily compliance backup to isolated account

    ProdAccount->>RDS: Create automated or manual snapshot
    RDS-->>ProdAccount: Snapshot available in us-east-1

    note over ProdAccount,Backup: Share snapshot cross-account

    ProdAccount->>ProdSnap: Modify snapshot attributes - share with backup account
    Backup->>ProdSnap: Copy shared snapshot to backup account
    ProdSnap-->>BackupSnap: Snapshot copied and now owned by backup account
    BackupSnap->>BackupSnap: Encrypt with backup account KMS key

    note over BackupSnap,Glacier: Optional - export to S3 for long-term archival

    BackupSnap->>BackupSnap: Export snapshot to S3 (Parquet format)
    BackupSnap->>Glacier: Transition to Glacier after 30 days
    note over Glacier: Glacier retention 7 years for compliance (HIPAA SOC2)

    note over Backup,Glacier: Backup account has no access to production resources.<br/>Even if prod account is compromised backup is safe.<br/>To restore - copy snapshot back to prod account or new account.
```

---

## 13. Velero — K8s StatefulSet PVC Backup and Restore

```mermaid
sequenceDiagram
    participant Velero as Velero Backup Controller
    participant K8s as Kubernetes API
    participant PVC as PostgreSQL PVC (EBS volume)
    participant S3 as S3 Backup Bucket
    participant EBS as AWS EBS Snapshot
    participant NewK8s as Target K8s Cluster (restore)

    note over Velero,EBS: Scheduled backup of database namespace

    Velero->>K8s: Get all resources in namespace=databases
    K8s-->>Velero: StatefulSet ConfigMap Secret PVC Service
    Velero->>EBS: Create EBS snapshot of PostgreSQL PVC
    EBS-->>Velero: Snapshot snap-xxx in progress
    Velero->>S3: Upload K8s resource manifests as JSON
    EBS-->>Velero: Snapshot completed

    note over Velero,NewK8s: Restore to new cluster (DR scenario)

    Velero->>NewK8s: Install Velero with same S3 backend
    NewK8s->>Velero: Sync backup metadata from S3
    Velero-->>NewK8s: Backups available

    Velero->>NewK8s: Restore namespace=databases from backup
    NewK8s->>K8s: Create StatefulSet Service ConfigMap Secret
    NewK8s->>EBS: Create new EBS volume from snapshot
    EBS-->>NewK8s: Volume available
    NewK8s->>PVC: Bind new PVC to restored volume
    NewK8s->>NewK8s: PostgreSQL pod starts with restored data

    note over Velero,NewK8s: Full K8s database restore including persistent data.<br/>RTO = time for EBS snapshot restore + pod startup (5-15 min).
```

---

## 14. Aurora Serverless v2 — Connection Scaling with RDS Proxy

```mermaid
sequenceDiagram
    participant Lambda as Lambda Functions (0 to 1000 instances)
    participant Proxy as RDS Proxy
    participant Pool as RDS Proxy Connection Pool
    participant ASv2 as Aurora Serverless v2
    participant Monitor as CloudWatch Metrics

    note over Lambda,Monitor: Lambda cold starts cause connection storms

    Lambda->>Proxy: 500 Lambda instances start simultaneously
    Proxy->>Proxy: Multiplex 500 Lambda connections into pool of 20 DB connections
    Proxy->>ASv2: 20 persistent connections to Aurora
    ASv2-->>Proxy: 20 connections ready
    Proxy-->>Lambda: All 500 Lambdas get connection from proxy pool

    note over ASv2,Monitor: Aurora Serverless scales compute

    Monitor->>ASv2: CPU at 90 percent - scale up needed
    ASv2->>ASv2: Scale from 2 ACU to 8 ACU (seconds - no restart)
    ASv2-->>Monitor: Capacity increased
    Proxy->>ASv2: Existing connections still valid - no reconnect needed

    note over Lambda,Monitor: Traffic drops - scale down

    Lambda->>Lambda: Invocations drop to 10 active
    Proxy->>Pool: Release idle connections
    Monitor->>ASv2: Scale down to 1 ACU after stabilization period
    note over ASv2: Serverless v2 minimum is 0.5 ACU - always ready, no cold start
```

---

## Summary — Migration and DR Quick Reference

| Scenario | Key Steps | Expected Downtime | Tools |
|----------|-----------|------------------|-------|
| On-prem PG to Aurora | Full load + CDC + cutover | 0 to 5 min | AWS DMS |
| Oracle to Aurora PG | SCT schema conversion + DMS | 0 to 5 min | SCT + DMS |
| MySQL to Aurora MySQL | Snapshot restore + DMS CDC | 0 to 2 min | Snapshot + DMS |
| SQL Server to Aurora PG | SCT + DMS + app changes | 1 to 4 hours | SCT + DMS |
| K8s PG backup to S3 | pg_dump via CronJob | None | pg_dump + aws s3 |
| Aurora Multi-AZ failover | Automatic | 20 to 60 sec | Built-in |
| PITR restore | Launch new instance from WAL | None (new instance) | RDS PITR |
| Aurora Global failover | Promote secondary | Under 1 min | aws rds failover-global-cluster |
| K8s rolling migration | Init container runs Flyway | 0 (rolling) | Flyway + init container |
| Read replica promotion | Manual promote + DNS update | 5 to 15 min | aws rds promote-read-replica |
| Cross-account snapshot | Share + copy snapshot | None | RDS snapshot share |
| Velero K8s restore | Apply backup to new cluster | 5 to 15 min | Velero + EBS snapshots |
