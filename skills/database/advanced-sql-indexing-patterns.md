# Advanced SQL Execution, Cache Synchronization & PostgreSQL Indexing Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ADVANCED SQL, CACHE SYNC & POSTGRESQL INDEXING PATTERNS                     ║
║  Cache invalidation lifecycle, Odoo 18 SQL() builder, partial unique         ║
║  indexes, and high-performance PostgreSQL tuning via _auto_init()            ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

When the Odoo ORM is too slow for bulk database mutations, or when advanced PostgreSQL constraints are required, developers must interact directly with SQL. 

Executing raw SQL incorrectly introduces two fatal production bugs:
1. **Cache Desynchronization (Stale Data)**: The ORM serves old values from memory because it is unaware of SQL changes.
2. **Missing ORM Writes**: Raw SQL queries miss pending in-memory ORM changes because they were not flushed to PostgreSQL.

This guide provides proven patterns for:
- Synchronizing the ORM cache around raw SQL queries.
- Constructing injection-proof queries using the **Odoo 18 `SQL()` builder**.
- Injected **Partial Unique Indexes** and **Trigram Indexes** via `_auto_init()`.

---

## 1. Raw SQL & Cache Synchronization Lifecycle

Whenever executing raw SQL `SELECT` or `UPDATE`, you **must** follow the 3-step synchronization lifecycle:

```
[ In-Memory ORM Changes ]
           │
           ▼ (Step 1: flush_model / flush_all)
[ PostgreSQL Database ] ──► (Step 2: cr.execute) ──► RETURNING id
           │
           ▼ (Step 3: invalidate_recordset & modified)
[ Cleared Cache & Recomputed Fields ]
```

### Complete Pattern: Safe Raw SQL Bulk Update

```python
from odoo import api, fields, models


class HeavyDataRecord(models.Model):
    _name = "heavy.data.record"
    _description = "Heavy Data Record"

    state = fields.Selection([("draft", "Draft"), ("done", "Done")], default="draft")
    processed_date = fields.Datetime()
    is_active = fields.Boolean(default=True)

    @api.model
    def bulk_mark_as_done(self, partner_id: int):
        """Perform a fast bulk SQL update while keeping ORM cache 100% consistent."""
        model_env = self.env["heavy.data.record"]

        # STEP 1: Flush pending ORM memory changes to PostgreSQL
        # Ensures raw SQL sees all writes previously made in this transaction
        model_env.flush_model(["state", "processed_date"])

        # STEP 2: Execute raw SQL with RETURNING clause
        # Always return the affected IDs so the ORM knows exactly what changed
        self.env.cr.execute(
            """
            UPDATE heavy_data_record
            SET state = 'done',
                processed_date = NOW() AT TIME ZONE 'UTC'
            WHERE partner_id = %s
              AND state = 'draft'
            RETURNING id
            """,
            [partner_id],
        )
        updated_ids = [row[0] for row in self.env.cr.fetchall()]

        if not updated_ids:
            return 0

        # STEP 3: Invalidate cache for the modified records
        # Prevents the ORM from returning stale 'draft' values from RAM
        records = model_env.browse(updated_ids)
        records.invalidate_recordset(["state", "processed_date"])

        # STEP 4: Notify the framework that fields were modified
        # Triggers recomputation of any stored/computed fields that depend on 'state'
        records.modified(["state", "processed_date"])

        return len(updated_ids)
```

### Invalidation & Flush Level Reference

| Method | Scope | Best Used When |
| :--- | :--- | :--- |
| `flush_model(['field_name'])` | Target model fields | Before running SQL `SELECT` on specific fields |
| `env.flush_all()` | Entire environment | Before massive complex multi-table queries |
| `invalidate_recordset(['field_name'])` | Specific record IDs | After SQL `UPDATE ... RETURNING id` |
| `invalidate_model(['field_name'])` | All records of model | When affected record IDs are unknown |
| `records.modified(['field_name'])` | Dependent computed fields | After any direct database write |

---

## 2. Odoo 18 Safe SQL Query Builder (`odoo.tools.SQL`)

In Odoo 18+, raw Python string interpolation and tuple formatting are deprecated in favor of `odoo.tools.SQL`. The wrapper prevents SQL injections, safely escapes identifiers (table/column names), and supports composable query joins.

### Basic Odoo 18 `SQL()` Usage

```python
from odoo import api, fields, models
from odoo.tools import SQL


class FleetMaintenanceLog(models.Model):
    _inherit = "fleet.vehicle"

    def get_aggregated_cost_sql(self, vehicle_ids: list[int], min_cost: float):
        query = SQL(
            """
            SELECT
                %s,
                sum(amount) AS total_cost
            FROM %s
            WHERE vehicle_id IN %s
              AND amount >= %s
            GROUP BY %s
            """,
            SQL.identifier("vehicle_id"),
            SQL.identifier("fleet_vehicle_cost"),
            tuple(vehicle_ids),
            min_cost,
            SQL.identifier("vehicle_id"),
        )
        self.env.cr.execute(query)
        return self.env.cr.dictfetchall()
```

### Dynamic Query Building with `SQL.join()`

Combine dynamic filters and clauses without messy string concatenation:

```python
    def search_custom_records(self, partner_id=None, state=None, limit=100):
        conditions = [SQL("active = %s", True)]

        if partner_id:
            conditions.append(SQL("partner_id = %s", partner_id))
        if state:
            conditions.append(SQL("state = %s", state))

        where_clause = SQL(" AND ").join(conditions)

        query = SQL(
            """
            SELECT id, name, state
            FROM %s
            WHERE %s
            ORDER BY id DESC
            LIMIT %s
            """,
            SQL.identifier(self._table),
            where_clause,
            limit,
        )
        self.env.cr.execute(query)
        return self.env.cr.dictfetchall()
```

---

## 3. Advanced PostgreSQL Constraints & Indexes via `_auto_init()`

Standard `_sql_constraints` in Odoo cannot handle conditional logic (e.g. `WHERE active = true`) or case-insensitive fields. Use `_auto_init()` to inject custom PostgreSQL indexes automatically during module updates.

### Pattern: Partial Unique Index & Case-Insensitive Index

```python
from odoo import api, fields, models, tools


class MiningEquipment(models.Model):
    _name = "mining.equipment"
    _description = "Mining Heavy Equipment Unit"

    name = fields.Char(required=True)
    unit_code = fields.Char(string="Unit Code", required=True)
    active = fields.Boolean(default=True)
    company_id = fields.Many2one("res.company", required=True, default=lambda self: self.env.company)

    def _auto_init(self):
        """Override _auto_init to create PostgreSQL indexes not supported by _sql_constraints."""
        res = super()._auto_init()

        # 1. Partial Unique Index:
        # unit_code must be unique per company, BUT ONLY for active records!
        # If a unit is archived, its unit_code can be reused.
        tools.create_index(
            self.env.cr,
            indexname="mining_equipment_unit_code_active_uniq",
            tablename=self._table,
            expressions=["lower(unit_code)", "company_id"],
            where="active = true",
            unique=True,
        )

        # 2. GIN Trigram Index (pg_trgm):
        # Enables instant fuzzy search (ILIKE '%query%') across millions of records.
        self.env.cr.execute("CREATE EXTENSION IF NOT EXISTS pg_trgm;")
        tools.create_index(
            self.env.cr,
            indexname="mining_equipment_name_trgm_idx",
            tablename=self._table,
            expressions=["name gin_trgm_ops"],
            method="gin",
        )

        return res
```

---

## 4. High-Churn Tables & Maintenance Considerations

Odoo contains high-churn tables with continuous row insertion and deletion:
- `bus_presence` (WebSocket presence heartbeats)
- `mail_message` & `mail_notification` (Chatter & emails)
- `queue_job` (Background task executions)
- `ir_logging` (System logs)

### PostgreSQL Autovacuum Tuning for High-Churn Tables
When high-churn tables become bloated, standard PostgreSQL vacuuming may lag behind. Recommended tuning in PostgreSQL for heavy Odoo workloads:

```sql
-- Tune autovacuum for high-write Odoo tables to prevent table bloat:
ALTER TABLE queue_job SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_vacuum_cost_limit = 1000
);

ALTER TABLE bus_presence SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 500
);
```

---

## 5. Summary & Best Practices Checklist

- [x] **Flush Before Raw SQL**: Call `flush_model()` before executing SQL queries that expect recent ORM modifications.
- [x] **Invalidate & Modified After Raw SQL**: Always call `invalidate_recordset()` and `records.modified()` to clear stale cache and recalculate dependent computed fields.
- [x] **Use `RETURNING id`**: Always append `RETURNING id` on raw `UPDATE` / `DELETE` statements so you know which specific records to invalidate.
- [x] **Use Odoo 18 `SQL()`**: Adopt `from odoo.tools import SQL` for all dynamic queries in v18+.
- [x] **Partial Indexes in `_auto_init()`**: Use `tools.create_index(..., where="active = true", unique=True)` for archive-safe uniqueness.

