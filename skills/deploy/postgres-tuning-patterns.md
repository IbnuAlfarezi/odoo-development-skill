# PostgreSQL 16 Performance Tuning Patterns for Odoo

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  POSTGRESQL 16 PERFORMANCE TUNING FOR ODOO                                  ║
║  Memory sizing (shared_buffers, work_mem), Checkpoints, Storage IOPS,       ║
║  Connection limits, Autovacuum optimization, and Maintenance queries         ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0 (PostgreSQL 14, 15, 16+)      ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo's ORM relies heavily on PostgreSQL for complex relational queries, joins, and transactions. Default PostgreSQL configurations are designed for machines with 512MB RAM and cause severe performance degradation on production workloads. Tuning `postgresql.conf` based on hardware resources is essential to prevent database locking, disk swapping, and slow query execution.

---

## 1. Production `postgresql.conf` Blueprint (Tuned for 8GB RAM Dedicated DB Server)

Place this file at `/etc/postgresql/16/main/postgresql.conf` or mount into your PostgreSQL container:

```ini
# =====================================================================
# 1. Connection Settings
# =====================================================================
listen_addresses = '*'
port = 5432
max_connections = 120                  # Total connections (Match Odoo db_maxconn * instances + headroom)

# =====================================================================
# 2. Memory Sizing (Calculated for 8GB RAM)
# =====================================================================
# shared_buffers: Dedicated buffer pool cache (25% of total server RAM)
shared_buffers = 2GB

# effective_cache_size: OS cache available for data caching (50% - 75% of RAM)
effective_cache_size = 6GB

# maintenance_work_mem: Used for CREATE INDEX, ALTER TABLE, VACUUM
maintenance_work_mem = 512MB

# work_mem: Memory used per sort/hash operation BEFORE writing to disk
# Keep reasonable (32MB - 64MB) because complex queries execute multiple operations concurrently
work_mem = 32MB

# wal_buffers: Memory for Write-Ahead Logs before disk flush
wal_buffers = 16MB

# =====================================================================
# 3. Checkpoints & WAL Performance (Smoothes Disk I/O Spikes)
# =====================================================================
# Target completion ratio across checkpoint interval (0.9 smooths write spikes)
checkpoint_completion_target = 0.9

# Size thresholds before triggering automatic checkpoint
max_wal_size = 4GB
min_wal_size = 1GB

# Checkpoint timeout interval
checkpoint_timeout = 15min

# =====================================================================
# 4. Storage & Query Planner (Optimized for SSD / NVMe)
# =====================================================================
# Default 4.0 assumes spinning magnetic HDD; SSD random seeks are almost identical to sequential
random_page_cost = 1.1
seq_page_cost = 1.0

# Number of concurrent disk I/O requests that can be handled simultaneously
effective_io_concurrency = 200

# Planner cost of fetching a tuple
cpu_tuple_cost = 0.01

# =====================================================================
# 5. Worker Processes & Parallelism
# =====================================================================
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2

# =====================================================================
# 6. Autovacuum Tuning for High-Volume Tables
# =====================================================================
# Odoo tables with high turnover (mail_message, bus_bus, stock_quant, ir_attachment)
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_analyze_threshold = 50
autovacuum_vacuum_scale_factor = 0.05   # Trigger vacuum when 5% of table changes
autovacuum_analyze_scale_factor = 0.02  # Trigger analyze when 2% changes
autovacuum_vacuum_cost_delay = 2ms
```

---

## 2. Memory Sizing Reference Table

| Total Server RAM | `shared_buffers` (25%) | `effective_cache_size` (75%) | `work_mem` | `maintenance_work_mem` |
| :---: | :---: | :---: | :---: | :---: |
| **4 GB** | 1 GB | 3 GB | 16 MB | 256 MB |
| **8 GB** | 2 GB | 6 GB | 32 MB | 512 MB |
| **16 GB** | 4 GB | 12 GB | 64 MB | 1024 MB |
| **32 GB** | 8 GB | 24 GB | 128 MB | 2048 MB |
| **64 GB** | 16 GB | 48 GB | 256 MB | 4096 MB |

---

## 3. Development vs Production PostgreSQL Settings

| Setting | Development Mode | Production Mode | Reason |
| :--- | :--- | :--- | :--- |
| **`max_connections`** | `50` | `120+` | Development only serves 1 local developer instance. |
| **`shared_buffers`** | `256MB` - `512MB` | `25% of RAM` | Save memory on development laptops. |
| **`synchronous_commit`**| `off` *(Optional for speed)* | **`on`** | `off` gives 3x faster test execution but risks losing transactions on crash. |
| **Port Exposure** | Expose `5432:5432` to host | **Internal network only** | Never expose port 5432 to public internet in production! |

---

## 4. Maintenance Queries for Database Admins

### A. Identify Bloated Tables:
```sql
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    n_dead_tup AS dead_tuples,
    n_live_tup AS live_tuples
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

### B. Manually Vacuum and Re-index after Heavy Imports:
```sql
-- Reclaim space and update query planner statistics
VACUUM (ANALYZE, VERBOSE) account_move_line;
VACUUM (ANALYZE, VERBOSE) stock_move_line;

-- Rebuild corrupted or fragmented B-Tree indexes concurrently without blocking reads
REINDEX TABLE CONCURRENTLY ir_attachment;
```

### C. Find Slow Queries Running in Odoo:
```sql
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE state != 'idle' AND (now() - pg_stat_activity.query_start) > interval '5 seconds'
ORDER BY duration DESC;
```

---

## 5. Security: Non-Superuser Role Creation (Official Odoo Security Rule)

Official Odoo documentation mandates that the PostgreSQL user used by Odoo **must NOT be a superuser**:

```sql
-- Connect as postgres administrative superuser:
-- 1. Create dedicated application role without administrative privileges
CREATE USER odoo_app WITH 
    NOSUPERUSER 
    NOCREATEDB 
    NOCREATEROLE 
    ENCRYPTED PASSWORD 'secure_production_password_here';

-- 2. Create production database owned by odoo_app
CREATE DATABASE production_company_db OWNER odoo_app;

-- 3. Grant schema permissions
GRANT ALL PRIVILEGES ON DATABASE production_company_db TO odoo_app;
```

---

## 6. Linux Host OS & Kernel Tuning (`sysctl.conf`)

For servers running PostgreSQL and Odoo multiprocessing workers:

### `/etc/sysctl.conf`:
```ini
# Reduce aggressive swapping (PostgreSQL performs poorly when memory pages swap to disk)
vm.swappiness = 10

# Increase maximum open file descriptors across system
fs.file-max = 2097152

# Increase network connection backlog queue for high concurrency bursts
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
```

### `/etc/security/limits.conf`:
```ini
# Increase user file descriptor limits (prevents 'Too many open files' socket errors)
odoo soft nofile 65535
odoo hard nofile 65535
postgres soft nofile 65535
postgres hard nofile 65535
```

