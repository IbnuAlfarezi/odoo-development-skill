# Concurrency & Locking Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  CONCURRENCY & TRANSACTION LOCKING PATTERNS                                  ║
║  Row-level locks, race condition prevention, savepoints, and transaction safety║
║  Essential for high-concurrency inventory, accounting, and sequence operations ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

In multi-user production environments, simultaneous requests to identical records cause race conditions (e.g. negative stock balances, duplicate sequence numbers, or inconsistent wallet deductions).

This guide provides proven patterns to protect data integrity using PostgreSQL row-level locks and transaction boundaries within Odoo.

---

## 1. The Golden Rule: Never Commit Manually

According to the official Odoo guidelines:
> **Never call `self.env.cr.commit()` inside business methods or controllers.**

### Why `cr.commit()` is Dangerous:
1. **Breaks Atomicity:** If an exception occurs later in the request, changes made before `cr.commit()` cannot be rolled back.
2. **Breaks Unit Tests:** Odoo tests run inside a single transaction that is rolled back at the end. Manual commits pollute the test database.
3. **Breaks ORM Cache:** The ORM cache can become desynchronized from the committed database state.

### Exception:
Manual commits are **only** permitted in standalone batch scripts, long-running CLI wizards, or dedicated workers that manage their own cursor lifecycle.

---

## 2. Preventing Race Conditions with Row-Level Locking

When two users read and update a balance or stock level simultaneously, a race condition occurs. To prevent this, lock the target row before reading it.

### Pattern: `SELECT ... FOR UPDATE`

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError


class AccountWallet(models.Model):
    _name = "account.wallet"
    _description = "User Wallet"

    partner_id = fields.Many2one("res.partner", required=True, index=True)
    balance = fields.Monetary(currency_field="currency_id", default=0.0)
    currency_id = fields.Many2one("res.currency", required=True)

    def deduct_balance(self, amount: float):
        self.ensure_one()
        if amount <= 0:
            raise UserError(_("Deduction amount must be positive."))

        # 1. Acquire row-level lock immediately (blocks concurrent transactions on this record)
        self.env.cr.execute(
            """
            SELECT balance
            FROM account_wallet
            WHERE id = %s
            FOR UPDATE NOWAIT
            """,
            [self.id],
        )

        # 2. Invalidate cache to read the freshest committed state
        self.invalidate_recordset(["balance"])

        # 3. Check and update safely
        if self.balance < amount:
            raise UserError(_("Insufficient balance."))

        self.balance -= amount
```

> [!TIP]
> Use `FOR UPDATE NOWAIT` to fail fast and notify the user immediately if another transaction is modifying the same record, rather than hanging indefinitely.

---

## 3. Savepoints for Partial Failure Recovery

When executing a batch of external API calls or operations where a single failure should not rollback the entire batch, use `savepoint()`:

```python
from odoo import api, models
import logging

_logger = logging.getLogger(__name__)


class BatchSyncService(models.AbstractModel):
    _name = "batch.sync.service"
    _description = "Batch Synchronization Service"

    @api.model
    def sync_records(self, record_ids: list[int]):
        records = self.env["external.sync.item"].browse(record_ids)
        success_count = 0

        for record in records:
            try:
                # Creates a PostgreSQL SAVEPOINT
                with self.env.cr.savepoint():
                    record._process_external_sync()
                    record.state = "synced"
                    success_count += 1
            except Exception as e:
                # If an error occurs inside the savepoint, only changes
                # for THIS record are rolled back. The loop continues safely.
                _logger.warning("Failed to sync item %s: %s", record.id, str(e))
                record.state = "failed"
                record.error_message = str(e)

        return success_count
```

---

## 4. Sequence Generation Locking Pattern

Custom document numbering must be strictly sequential without gaps or duplicates under concurrent load:

```python
class MiningProductionLog(models.Model):
    _name = "mining.production.log"
    _description = "Mining Production Log"

    name = fields.Char(string="Reference", default="/", copy=False, readonly=True)

    @api.model_create_multi
    def create(self, vals_list: list[dict]):
        for vals in vals_list:
            if vals.get("name", "/") == "/":
                # Lock sequence to prevent duplicate numbers under concurrency
                vals["name"] = self.env["ir.sequence"].next_by_code(
                    "mining.production.log"
                ) or "/"
        return super().create(vals_list)
```

---

## 5. Web Client Concurrency (`KeepLast`)

On the frontend, rapid user input or multiple search requests can resolve out-of-order. Odoo OWL provides the `KeepLast` utility to ensure only the latest request updates the UI:

```javascript
import { Component, useState } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";
import { KeepLast } from "@web/core/utils/concurrency";

export class SearchableList extends Component {
    static template = "my_module.SearchableList";

    setup() {
        this.orm = useService("orm");
        this.state = useState({ records: [], loading: false });
        this.keepLast = new KeepLast();
    }

    async onSearch(query) {
        this.state.loading = true;
        
        // KeepLast cancels or discards results from previous unresolved calls
        const results = await this.keepLast.add(
            this.orm.searchRead("my.model", [["name", "ilike", query]], ["name", "code"])
        );
        
        this.state.records = results;
        this.state.loading = false;
    }
}
```

---

## 6. Summary of Best Practices

1. **Always use savepoints** (`with self.env.cr.savepoint():`) when handling multi-item transactions where individual failures should be logged rather than aborting everything.
2. **Lock records before balance checks** using `SELECT ... FOR UPDATE` followed by `self.invalidate_recordset()`.
3. **Never call `cr.commit()`** inside standard models or controllers. Let Odoo handle transaction boundaries automatically.
4. **Use `KeepLast`** in OWL components when handling search-as-you-type or rapid asynchronous operations.

