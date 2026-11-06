# Backup & Disaster Recovery Patterns for Odoo

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO BACKUP & DISASTER RECOVERY PATTERNS                                    ║
║  Automated dual-component backups (PostgreSQL + Filestore), Retention rules, ║
║  Safe Restore procedures, and Staging Database Neutralization SQL            ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

A common disaster recovery pitfall in Odoo is backing up only the PostgreSQL database. In Odoo, attachments, reports, images, and binary documents are stored on the filesystem (`/var/lib/odoo/filestore/<db_name>`) rather than in database tables. **A database dump without its corresponding filestore will result in corrupted attachments and broken document links.**

A complete Odoo backup always consists of two synchronized components:
1. **Database Dump**: Generated via `pg_dump -Fc` (PostgreSQL custom archive format).
2. **Filestore Archive**: Gzip compressed tar of the directory `/var/lib/odoo/filestore/<db_name>`.

---

## 1. Automated Nightly Backup Script (`backup.sh`)

Place this script on your production host (e.g. `/usr/local/bin/odoo-backup.sh`) and execute via `cron`:

```bash
#!/usr/bin/env bash
# =====================================================================
# Odoo Automated Dual-Component Backup Script
# =====================================================================
set -eo pipefail

BACKUP_ROOT="/var/backups/odoo"
DATE=$(date +'%Y%m%d_%H%M%S')
DB_NAME="production_company_db"
CONTAINER_DB_NAME="odoo_prod_db"
FILESTORE_PATH="/var/lib/odoo/filestore/${DB_NAME}"
TEMP_DIR="${BACKUP_ROOT}/tmp_${DATE}"
FINAL_ARCHIVE="${BACKUP_ROOT}/odoo_backup_${DB_NAME}_${DATE}.tar.gz"

mkdir -p "${TEMP_DIR}" "${BACKUP_ROOT}/daily" "${BACKUP_ROOT}/weekly"

echo "[*] Step 1: Exporting PostgreSQL database..."
docker exec "${CONTAINER_DB_NAME}" pg_dump -U odoo -Fc "${DB_NAME}" > "${TEMP_DIR}/dump.sql"

echo "[*] Step 2: Archiving filestore..."
if [ -d "${FILESTORE_PATH}" ]; then
    tar -czf "${TEMP_DIR}/filestore.tar.gz" -C "${FILESTORE_PATH}" .
else
    echo "[!] Warning: Filestore path not found, creating empty archive."
    tar -czf "${TEMP_DIR}/filestore.tar.gz" --files-from /dev/null
fi

echo "[*] Step 3: Packaging master backup archive..."
tar -czf "${FINAL_ARCHIVE}" -C "${TEMP_DIR}" .
rm -rf "${TEMP_DIR}"

# Organize into daily rotation
mv "${FINAL_ARCHIVE}" "${BACKUP_ROOT}/daily/"
LATEST_BACKUP="${BACKUP_ROOT}/daily/odoo_backup_${DB_NAME}_${DATE}.tar.gz"

echo "[*] Step 4: Applying retention policies..."
# Keep 7 daily backups
find "${BACKUP_ROOT}/daily" -name "odoo_backup_*.tar.gz" -type f -mtime +7 -delete

# If Sunday, copy to weekly retention (Keep 4 weeks)
if [ "$(date +%u)" -eq 7 ]; then
    cp "${LATEST_BACKUP}" "${BACKUP_ROOT}/weekly/"
    find "${BACKUP_ROOT}/weekly" -name "odoo_backup_*.tar.gz" -type f -mtime +28 -delete
fi

# Optional Step 5: Sync offsite to AWS S3 or MinIO
# aws s3 cp "${LATEST_BACKUP}" "s3://my-company-odoo-backups/daily/"

echo "[✓] Backup completed successfully: ${LATEST_BACKUP}"
```

### Install Crontab Execution:
```bash
# Execute every night at 02:00 AM
0 2 * * * /usr/local/bin/odoo-backup.sh >> /var/log/odoo-backup.log 2>&1
```

---

## 2. Safe Database Restore Procedure (`restore.sh`)

To restore a backup into a new sandbox or disaster recovery target:

```bash
#!/usr/bin/env bash
set -eo pipefail

BACKUP_FILE="$1"
TARGET_DB="$2"
FILESTORE_ROOT="/var/lib/odoo/filestore"
TEMP_DIR="/tmp/odoo_restore_$$"

if [ -z "${BACKUP_FILE}" ] || [ -z "${TARGET_DB}" ]; then
    echo "Usage: $0 <path_to_backup.tar.gz> <target_db_name>"
    exit 1
fi

mkdir -p "${TEMP_DIR}" "${FILESTORE_ROOT}/${TARGET_DB}"

echo "[*] Step 1: Extracting master archive..."
tar -xzf "${BACKUP_FILE}" -C "${TEMP_DIR}"

echo "[*] Step 2: Creating database and restoring SQL dump..."
docker exec odoo_prod_db createdb -U odoo -O odoo "${TARGET_DB}"
docker exec -i odoo_prod_db pg_restore -U odoo -d "${TARGET_DB}" --no-owner < "${TEMP_DIR}/dump.sql"

echo "[*] Step 3: Restoring filestore attachments..."
if [ -f "${TEMP_DIR}/filestore.tar.gz" ]; then
    tar -xzf "${TEMP_DIR}/filestore.tar.gz" -C "${FILESTORE_ROOT}/${TARGET_DB}"
fi

rm -rf "${TEMP_DIR}"
echo "[✓] Database ${TARGET_DB} restored successfully!"
```

---

## 3. Mandatory Staging Neutralization (Prevent Accidental Customer Contact)

When restoring a **Production backup into a Development, Staging, or QA environment**, you **MUST neutralize** the database. Otherwise, scheduled crons might email real customers or charge active payment gateways!

Execute this SQL script immediately after restoring onto a staging database:

```sql
-- 1. Deactivate all automated cron jobs
UPDATE ir_cron 
SET active = FALSE;

-- 2. Deactivate outgoing mail servers (Prevents test emails sent to real clients)
UPDATE ir_mail_server 
SET active = FALSE;

-- 3. Reset mail server credentials
UPDATE ir_mail_server 
SET smtp_encryption = 'none', smtp_port = 25, smtp_host = 'localhost';

-- 4. Switch all payment providers to 'test' or 'disabled' mode
UPDATE payment_provider 
SET state = 'test';

-- 5. Update Web Base URL to staging domain
UPDATE ir_config_parameter 
SET value = 'https://staging-erp.mycompany.com' 
WHERE key = 'web.base.url';

-- 6. Freeze web base URL to prevent automatic overwrite by admin login
INSERT INTO ir_config_parameter (key, value)
VALUES ('web.base.url.freeze', 'True')
ON CONFLICT (key) DO UPDATE SET value = 'True';
```

