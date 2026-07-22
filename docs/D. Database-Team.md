## D. DATABASE TEAM – PRACTICAL SCHEMA, PARTITIONING & DISASTER DRILLS (DEEP DIVE)

---

### D1. Schema Design & Data Types (The Foundation That Cannot Change Later)
*Decision needed: Choosing the wrong data type now means a 12-hour migration later. We must get this 100% right.*

- **Primary Keys: UUID vs Serial/BigInt:**
  - **Decision:** Use **UUID (v4)** as the primary key for all core tables (`School`, `User`, `Curriculum`, `Subscription`).
  - **Why:** We are building a multi-tenant, distributed system. If we ever shard across multiple DB clusters (e.g., AP schools on Cluster 1, TG on Cluster 2), `SERIAL` (auto-increment) will cause collisions. UUIDs are globally unique.
  - **Implementation:** Use `django.db.models.UUIDField(default=uuid.uuid4, editable=False, primary_key=True)`. 
  - *Performance Caveat:* UUIDs are 16 bytes vs 4 bytes for an int, making indexes larger. **Mitigation:** Use `uuid_generate_v7()` (time-ordered UUIDs) instead of v4 to maintain B-tree insertion locality and prevent index fragmentation. We will install the `pgcrypto` extension and define a custom default in Django.
- **Foreign Keys & Referential Integrity (The `on_delete` Debate):**
  - **Decision:** Use `PROTECT` or `RESTRICT` for `School → User` relationships. We **cannot** allow a `CASCADE` delete—if a Chairman accidentally deletes a school, we must require manual super-admin intervention.
- **State & Enum Fields (Use PostgreSQL ENUMs vs Checks):**
  - **Decision:** For fixed values like `state` ('AP', 'TG') and `subscription_status` ('PENDING', 'ACTIVE', 'EXPIRED', 'GRACE_PERIOD'), use **PostgreSQL `CREATE TYPE` ENUMs** instead of `CharField` with `choices`.
  - **Why:** ENUMs are stored as 4-byte integers internally (space-efficient) and throw explicit errors if an invalid string attempts insertion.
  - **Django Implementation:** Use `django.contrib.postgres.fields.ArrayField` or third-party `django-enumfield`, but simpler: define the raw SQL in a migration (`RunSQL`).
- **JSONB vs Relational for Simulation Config:**
  - `simulation_config_json` in the `Curriculum` table holds coefficients, sliders, axis limits. 
  - **Decision:** Use `JSONB` (not plain JSON). 
  - **Why:** `JSONB` is binary, indexable, and supports `->>`, `@>` operators. We will add a **GIN index** on this column for future metadata querying (e.g., "Find all topics where `coefficient > 5`"). 

---

### D2. Indexing Strategy (The "Slow Query" Prevention Manifesto)
*Decision needed: Indexes speed reads but slow writes. We must pick only the essential ones.*

- **The Golden Index Set (Must-Haves):**
  ```sql
  -- 1. Subdomain lookup (fastest possible)
  CREATE UNIQUE INDEX CONCURRENTLY idx_school_subdomain ON school (subdomain);
  
  -- 2. Curriculum Tree traversal (composite)
  CREATE INDEX CONCURRENTLY idx_curriculum_hierarchy ON curriculum (state, class_level, subject, lesson_slug);
  
  -- 3. User lookup by email for login
  CREATE INDEX CONCURRENTLY idx_user_email ON "user" (email);
  
  -- 4. Subscription lookups for celery expiry cron
  CREATE INDEX CONCURRENTLY idx_subscription_status_expiry ON subscription (status, expiry_date) WHERE status = 'ACTIVE';
  ```
- **Partial Indexes (Huge Space Savers):** 
  - We only query active schools 95% of the time. 
  - **Decision:** Create a partial index on `school` where `is_deleted = False`. This drastically shrinks the index size.
- **Concurrent Indexing (Zero Downtime):**
  - **Hard Rule:** Every `CREATE INDEX` or `DROP INDEX` in production **MUST** use the `CONCURRENTLY` keyword. Without it, the table is locked for writes for the entire indexing duration (potentially hours).
  - *Git Hook:* Add a pre-migration hook that scans for `CREATE INDEX` without `CONCURRENTLY` and fails the CI build.

---

### D3. Partitioning Strategy (The "50 Million Rows/Year" Survival Guide)
*Decision needed: We must partition the `TeacherActivity` table. Without this, the DB dies in Year 2.*

- **Partition Key Choice:** **RANGE** on `created_at` (Monthly partitions).
- **Partition Creation Automation (The "Future-Proof" Script):**
  - Since Django's ORM does not natively support partition creation, we write a **native PostgreSQL function + trigger** OR use a Celery beat task running on the 1st of every month.
  - **Exact SQL (Run via Django `RunSQL` migration):**
    ```sql
    -- Create the main partitioned table
    CREATE TABLE teacher_activity (
        id UUID DEFAULT gen_random_uuid() NOT NULL,
        school_id UUID NOT NULL REFERENCES school(id),
        user_id UUID NOT NULL REFERENCES "user"(id),
        topic_id UUID NOT NULL REFERENCES curriculum(id),
        annotation_json JSONB,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
    ) PARTITION BY RANGE (created_at);

    -- Create a partition for the current month (e.g., August 2026)
    CREATE TABLE teacher_activity_2026_08 PARTITION OF teacher_activity
        FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

    -- Index each partition individually!
    CREATE INDEX idx_activity_school_topic_2026_08 ON teacher_activity_2026_08 (school_id, topic_id);
    ```
  - **Partition Pruning:** The query `WHERE created_at BETWEEN '2026-08-15' AND '2026-08-20'` will automatically scan *only* the `2026_08` partition due to constraint exclusion. **Ensure `constraint_exclusion = on` in `postgresql.conf`**.
- **Handling Late-Arriving Data (Clock Skew):**
  - If a server's clock is 1 minute behind, a write intended for 11:59 PM might fall into the next partition. 
  - **Decision:** We set the partition creation cron to create partitions **3 months in advance** (e.g., in August, create partitions for Sep, Oct, Nov). This absorbs any clock drift or delayed writes.

---

### D4. Connection Pooling (PgBouncer Deep Configuration)
*Decision needed: Django's `CONN_MAX_AGE` + PgBouncer in `transaction` mode is a classic trap. We must agree on the exact settings.*

- **Pooling Mode Decision:** **Transaction Pooling**. 
  - **Why not Session?** Session pooling holds a connection open for the entire HTTP request duration, which scales poorly. Transaction pooling releases the connection back to the pool after every DB transaction (`commit`/`rollback`).
- **The `PREPARE` Statement Trap:** Django uses prepared statements by default. With transaction pooling, `PREPARE` statements fail because the connection changes between transactions.
  - **Mitigation:** Set `DISABLE_PREPARED_STATEMENTS = True` in Django settings (Django 4.2+). Alternatively, set `pool_mode = transaction` and set `server_reset_query = DISCARD ALL` in PgBouncer config.
- **Pool Size Calculation (Exact Math):**
  - Assume 50 concurrent Django application servers (Gunicorn workers). Each worker holds 1 connection. 
  - **Decision:** Set PgBouncer `pool_size = 200` (leaves room for spikes). Set `max_client_conn = 1000`. 
  - *Alert:* If connections exceed 200, PgBouncer queues requests instead of failing. If queue time exceeds 5 seconds, we must scale out the DB.

---

### D5. Backup Strategy (Point-in-Time Recovery - PITR)
*Decision needed: If a dev accidentally drops a table at 10:15 AM, we must restore to exactly 10:14 AM without losing the last 10 minutes of payments.*

- **WAL Archiving (The Non-Negotiable):**
  - Enable `archive_mode = on` and `archive_command = 'test ! -f /mnt/backups/wal/%f && cp %p /mnt/backups/wal/%f'`.
  - **Decision:** Ship WALs to **AWS S3** (or equivalent) using `wal-e` or `pg_backrest` every **5 minutes**.
- **Full Backup (Base Backup) Schedule:**
  - Take a full `pg_basebackup` **every 24 hours at 2:00 AM** (off-peak).
  - Retain 30 daily backups, 12 monthly backups.
- **Recovery Runbook (The "10:14 AM" Drill):**
  - **Stop** the Django application to prevent new writes.
  - **Restore** the latest base backup.
  - **Apply WAL files** from S3 sequentially up to `2026-08-20 10:14:00` using `recovery_target_time`.
  - **Promote** the DB (`SELECT pg_promote()`).
  - **Restart** the application.
  - **Expected RTO:** 15 minutes. **Expected RPO:** < 5 minutes (as WALs ship every 5 mins).

---

### D6. Disaster Drills (The "Fire Drill" Protocol)
*Decision needed: We cannot wait for a real outage to test our backups. We must schedule and document actual drills.*

- **Mandatory Monthly Drill (Weekend 6 AM):**
  1. **Simulate Data Corruption:** A DBA manually runs `DELETE FROM teacher_activity WHERE created_at > NOW() - interval '1 hour';` on a staging replica.
  2. **Restore** that replica from the latest PITR using the exact runbook.
  3. **Measure Time-to-Restore:** If it takes > 20 minutes, we optimize the WAL replay speed (increase `wal_buffers`, tune `checkpoint_completion_target`).
  4. **Promote and Verify:** Run `SELECT COUNT(*)` to ensure the restored data matches expected values.
- **Failover to Read-Replica (High Availability):**
  - We will maintain a **Synchronous Streaming Replica** in a different availability zone (AZ).
  - **Decision:** If primary AZ goes down, we manually (or via AWS RDS automatic) promote the replica. 
  - *Warning:* Synchronous replication adds 5-10ms latency per write. We accept this for zero data loss.

---

### D7. Vacuuming & Performance Maintenance (Preventing "Bloat")
*Decision needed: Autovacuum defaults are too conservative for our write-heavy workload.*

- **Autovacuum Tuning (Explicit Settings):**
  - `autovacuum_vacuum_scale_factor = 0.01` (Default is 0.2, which means 20% of the table must be dead before vacuum runs—way too high for our 50M row table).
  - `autovacuum_vacuum_threshold = 5000`.
  - `autovacuum_naptime = 15s` (Check more frequently).
- **Monitoring Query (Alert Threshold):**
  - Run this every hour:
    ```sql
    SELECT schemaname, tablename, n_dead_tup, n_live_tup, 
           round(n_dead_tup::numeric / n_live_tup * 100, 2) AS dead_pct 
    FROM pg_stat_user_tables 
    WHERE n_dead_tup > 10000;
    ```
  - **Decision:** If `dead_pct > 10%`, trigger a critical alert to the DBA to run `VACUUM ANALYZE` manually.
- **`pg_repack` for Zero-Downtime Index Rebuilds:**
  - When indexes bloat over time, `REINDEX` locks the table. 
  - **Decision:** Install `pg_repack` extension to rebuild indexes and reclaim space online without locking.

---

### D8. Migration Safety (The "No Downtime" Deployment Rule)
*Decision needed: We cannot take the app offline for DB schema changes.*

- **Backward-Compatible Migrations (The Golden Rule):**
  - **Rule 1:** *Never* rename a column in a single migration. Do it in 3 steps:
    1. Add new column (allow null).
    2. Backfill data.
    3. Drop old column (only after the code no longer references it).
  - **Rule 2:** *Never* add a `NOT NULL` constraint without adding a `DEFAULT` or backfilling first.
- **Django Migration `--plan` & CI Check:**
  - Before any migration runs in production, the CI pipeline must output `python manage.py sqlmigrate --plan` and require a Senior DB Admin to review the raw SQL.
  - **Decision:** We enforce `MIGRATION_MODULES` and use `migrate --fake` for rollbacks only in emergencies.
- **Large Data Backfills (The Chunking Strategy):**
  - If we need to update 1 million rows (e.g., adding a new `school.timezone` field), **do NOT** use a single `UPDATE` statement—it will lock the table for 5 minutes.
  - **Decision:** Use `django-bulk-update` or a custom Celery task that updates rows in batches of 1000 (`LIMIT 1000 OFFSET X`) with a 100ms sleep between batches to allow other queries to sneak in.

---

### Database Team Sign-Off Checklist (To Leave the Meeting With)

| # | Action Item | Owner | Deadline |
| :--- | :--- | :--- | :--- |
| 1 | Switch all Primary Keys to `UUID` with `uuid_generate_v7()` (install `pgcrypto`). Write migration plan. | DB Lead | Sprint 1 |
| 2 | Create the **Monthly Partitioning** DDL script for `TeacherActivity` and schedule the Cron/Celery Beat task for partition creation. | DB Lead | Sprint 2 |
| 3 | Write the **Index Creation** migration with `CONCURRENTLY` and enforce the pre-migration CI check. | DB Lead | Sprint 1 |
| 4 | Configure PgBouncer with `transaction` pooling, set `pool_size=200`, and test Django connectivity with `DISABLE_PREPARED_STATEMENTS=True`. | DevOps / DB | Sprint 1 |
| 5 | Set up **WAL Archiving to S3** and schedule the daily `pg_basebackup`. Document the PITR runbook. | DevOps / DB | Sprint 3 |
| 6 | Schedule the **First Disaster Drill** (simulate corruption, test RTO/RPO) on a Staging replica. Log the time taken. | DB Lead | End of Sprint 4 |
| 7 | Tune `autovacuum` parameters (`scale_factor=0.01`) and set up monitoring alerts for dead tuples > 10%. | DB Lead | Sprint 2 |
| 8 | Enforce the "3-Step Migration" rule (Add -> Backfill -> Drop) in the Developer Code Review checklist. | CTO / Tech Lead | Immediate |
