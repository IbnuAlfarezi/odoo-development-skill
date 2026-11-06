# Odoo Server Configuration (`odoo.conf`) Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO SERVER CONFIGURATION (odoo.conf) PATTERNS                              ║
║  Side-by-side comparison of Production vs Development configurations,        ║
║  Multiprocessing worker formulas, Memory thresholds, and Security parameters ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

The `odoo.conf` file controls runtime behavior, process management, database connectivity, and security boundaries. The configuration must be tailored specifically for either **Production** (stability, concurrency, memory capping, zero-trust) or **Development** (hot-reload, single-threading for debuggers, relaxed database access).

---

## 1. Production Mode Configuration (`odoo.prod.conf`)

In production, Odoo **MUST** run in multiprocessing mode (`workers > 0`) behind a reverse proxy (`proxy_mode = True`) with database management disabled (`list_db = False`).

```ini
[options]
; =====================================================================
; 1. Database Connection & Connection Pooling
; =====================================================================
db_host = postgres
db_port = 5432
db_user = odoo_app                     ; Dedicated NON-SUPERUSER for security (never use 'postgres')
db_password = secure_production_password_here
db_name = production_company_db
db_maxconn = 64
db_sslmode = prefer                    ; 'prefer', 'require', or 'verify-full'
db_template = template1
; db_replica_host = postgres_replica   ; Optional: Read-replica for heavy reporting offloading
; db_replica_port = 5432

; =====================================================================
; 2. Security & Database Access Control (Official Odoo Hardening)
; =====================================================================
; Disable public database selector (/web/database/manager)
; Warning: Strongly recommended by official Odoo docs for any internet-facing system!
list_db = False

; Lock domain routing strictly to the target company database
dbfilter = ^production_company_db$

; Secure master password hashed with PBKDF2-SHA512
; Generate via: python3 -c "import passlib.hash; print(passlib.hash.pbkdf2_sha512.hash('your_master_secret'))"
admin_passwd = $pbkdf2-sha512$25000$P98B...

; Do NOT install demo data in production under any circumstance
without_demo = True

; =====================================================================
; 3. Multiprocessing & Resource Limits (Tuned for 4 CPU / 8GB RAM)
; =====================================================================
; Workers = (2 * CPU Cores) + 1
workers = 9

; Dedicated threads for background cron jobs
max_cron_threads = 1

; Timeouts in seconds (Prevents zombie queries from starving the pool)
limit_time_cpu = 600
limit_time_real = 1200
limit_time_real_cron = -1

; Memory Thresholds in Bytes (Prevents runaway leaks from crashing server)
; soft = ~640MB * workers | hard = ~800MB * workers
limit_memory_soft = 5905580032
limit_memory_hard = 7516192768
limit_request = 8192

; =====================================================================
; 4. Reverse Proxy & File Offloading (X-Accel / X-Sendfile)
; =====================================================================
; CRITICAL: Must be True behind Nginx to trust X-Forwarded-* client headers
proxy_mode = True
http_interface = 0.0.0.0
http_port = 8069
gevent_port = 8072

; Performance: Delegate attachment streaming directly to Nginx (X-Accel-Redirect)
; Saves Python worker memory by letting Nginx stream large documents/attachments
x_sendfile = True

; =====================================================================
; 5. Logging, Addons & Storage
; =====================================================================
logfile = /var/log/odoo/odoo-server.log
log_level = info
addons_path = /usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons
data_dir = /var/lib/odoo
```

---

## 2. Development Mode Configuration (`odoo.dev.conf`)

In development, Odoo **MUST** run in single-threaded mode (`workers = 0`) to allow interactive breakpoints (`pdb`/`ipdb`/`debugpy`), disable timeouts, and enable hot-reloading:

```ini
[options]
; =====================================================================
; 1. Database Connection (Local / Docker Desktop)
; =====================================================================
db_host = localhost
db_port = 5432
db_user = odoo
db_password = odoo
db_maxconn = 32

; =====================================================================
; 2. Developer Convenience & Database Freedom
; =====================================================================
; Enable public database manager for quick create/drop/duplicate
list_db = True
dbfilter = .*
admin_passwd = admin

; =====================================================================
; 3. CRITICAL DEBUGGING SETTINGS: Single-Threaded Mode
; =====================================================================
; MUST be 0! Multi-worker mode forks child processes that detach from the
; terminal, completely breaking pdb, ipdb, and IDE debuggers.
workers = 0
max_cron_threads = 0

; Disable timeouts so debugger breakpoints do not trigger 504 timeouts
limit_time_cpu = 0
limit_time_real = 0

; =====================================================================
; 4. Hot-Reload & Developer Tooling Flags
; =====================================================================
; - reload: Auto-restarts server upon Python file save
; - qweb: Recompiles QWeb templates without module upgrade
; - xml: Reloads XML view definitions directly from disk
; - werkzeug: Displays interactive traceback console on HTTP exceptions
dev_mode = reload,qweb,xml,werkzeug

; =====================================================================
; 5. Logging & Addons
; =====================================================================
log_level = debug
addons_path = /path/to/odoo/addons,/path/to/custom_addons
data_dir = ~/.local/share/Odoo
```

---

## 3. Side-by-Side Parameter Matrix

| Parameter | Development Mode | Production Mode | Why It Matters |
| :--- | :--- | :--- | :--- |
| **`workers`** | **`0`** | **`(2 * CPU) + 1`** | `0` allows step-debugging (`pdb`/`debugpy`). `>0` enables multi-core request handling. |
| **`dev_mode`** | `reload,qweb,xml,werkzeug` | *Unset / False* | Hot-reload burns CPU checks in production and exposes Werkzeug security holes. |
| **`proxy_mode`** | `False` | **`True`** | Required behind Nginx so `request.httprequest.remote_addr` returns client IP, not `127.0.0.1`. |
| **`limit_time_real`** | `0` (Disabled) | `1200` (Enforced) | Protects production from long-running or deadlocked queries. |
| **`limit_memory_hard`**| `0` (Unlimited) | `800MB * workers` | Hard-kills runaway worker processes before system OOM killer crashes PostgreSQL. |
| **`list_db`** | `True` | **`False`** | Prevents unauthorized visitors from seeing database names or creating test databases. |
| **`dbfilter`** | `.*` (All databases) | `^company_db$` | Prevents sessions from crossing into other tenant databases on multi-domain hosts. |
| **`log_level`** | `debug` | `info` / `warn` | `debug` writes gigabytes of disk I/O under production traffic. |
| **`admin_passwd`** | Plain text (`admin`) | **PBKDF2-SHA512 Hash** | Protects the master database management endpoint from brute-force attacks. |

---

## 4. Environment Variables Overrides

Any `odoo.conf` parameter can be overridden using environment variables or CLI flags:

```bash
# Path to configuration file
export ODOO_RC="/etc/odoo/odoo.conf"

# Common Docker container overrides
export HOST="postgres"
export USER="odoo"
export PASSWORD="secure_password"

# CLI flag overrides
python3 odoo-bin -c odoo.prod.conf --workers=5 --max-cron-threads=1
```

