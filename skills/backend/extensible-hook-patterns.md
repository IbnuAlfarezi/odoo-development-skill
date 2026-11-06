# Extensible Hook & ORM Batching Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  EXTENSIBLE HOOK & ORM BATCHING PATTERNS                                     ║
║  Clean OCA hook architecture, _prepare_* value methods, and ORM prefetching  ║
║  Ensures clean module inheritance and eliminates N+1 query performance leaks ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

In professional Odoo development adhering to Odoo Community Association (OCA) standards, custom modules must be designed for **extensibility** and **performance**:

1. **Extensible Hook Pattern**: Never hardcode values inside monolithic `create()` or action methods. Always isolate value dictionaries into dedicated `_prepare_<target>_vals()` methods.
2. **Lifecycle Hooks**: Break multi-step operations into `_before_*()`, `_action_*()`, and `_after_*()` hooks.
3. **ORM Prefetching & Batching**: Leverage Odoo's batch prefetch mechanism and chunking to eliminate N+1 queries and prevent Out-Of-Memory (OOM) failures.

---

## 1. The `_prepare_*` Value Hook Pattern

When one document generates another (e.g. Sales Order $\rightarrow$ Invoice, Maintenance Request $\rightarrow$ Work Order, Mining Shift $\rightarrow$ Production Log), isolate the dictionary preparation.

### Base Module Implementation

```python
from odoo import api, fields, models


class FleetInspection(models.Model):
    _name = "fleet.inspection"
    _description = "Fleet Vehicle Inspection"

    vehicle_id = fields.Many2one("fleet.vehicle", required=True)
    inspector_id = fields.Many2one("res.users", default=lambda self: self.env.user)
    date = fields.Date(default=fields.Date.context_today)
    odometer = fields.Float(string="Odometer Reading")
    notes = fields.Text()

    def _prepare_work_order_vals(self) -> dict:
        """Hook method to prepare dictionary values for creating a maintenance work order.
        Subordinate modules should override this method to inject additional fields.
        """
        self.ensure_one()
        return {
            "vehicle_id": self.vehicle_id.id,
            "origin_inspection_id": self.id,
            "description": f"Maintenance triggered by inspection on {self.date}",
            "priority": "1",
            "date_planned": fields.Datetime.now(),
        }

    def action_create_work_order(self):
        """Action method orchestrating the creation."""
        orders = self.env["fleet.work.order"]
        for inspection in self:
            vals = inspection._prepare_work_order_vals()
            order = self.env["fleet.work.order"].create(vals)
            orders |= order
        return orders
```

### Inheriting Module (Clean Extension Without Monkey Patching)

```python
class FleetInspection(models.Model):
    _inherit = "fleet.inspection"

    hour_meter = fields.Float(string="Engine Hour Meter")
    severity_level = fields.Selection([
        ("low", "Low"),
        ("critical", "Critical")
    ], default="low")

    def _prepare_work_order_vals(self) -> dict:
        """Extend values cleanly without touching creation logic."""
        vals = super()._prepare_work_order_vals()
        vals.update({
            "hour_meter": self.hour_meter,
            "is_critical": self.severity_level == "critical",
        })
        return vals
```

---

## 2. Multi-Step Lifecycle Hooks

Avoid placing 100 lines of complex logic inside a single button action. Break the flow into distinct lifecycle hooks:

```python
from odoo import _, api, models
from odoo.exceptions import UserError


class MiningHaulingSession(models.Model):
    _name = "mining.hauling.session"
    _description = "Mining Hauling Session"

    state = fields.Selection([
        ("draft", "Draft"),
        ("in_progress", "In Progress"),
        ("closed", "Closed")
    ], default="draft")

    def action_close_session(self):
        """Orchestrator pattern using lifecycle hooks."""
        for session in self:
            session._before_close_session()
            session._action_close_session()
            session._after_close_session()
        return True

    def _before_close_session(self):
        """Validation hook."""
        self.ensure_one()
        if self.state != "in_progress":
            raise UserError(_("Only active sessions can be closed."))

    def _action_close_session(self):
        """Core state change and financial/inventory posting."""
        self.ensure_one()
        self.write({"state": "closed"})

    def _after_close_session(self):
        """Notification, audit logging, and external sync hook."""
        self.ensure_one()
        self.message_post(body=_("Hauling session closed successfully."))
```

---

## 3. ORM Prefetching & Eliminating N+1 Queries

Odoo's ORM automatically prefetches fields for all records in the same recordset. Breaking this recordset causes catastrophic N+1 query performance hits.

### ❌ Anti-Pattern: Individual Record Browsing in a Loop
```python
# BAD: Memicu 1 query database terpisah untuk setiap record_id!
for partner_id in partner_ids:
    partner = self.env["res.partner"].browse(partner_id)
    print(partner.name)   # N queries to PostgreSQL
```

### ✅ Best Practice: Batch Browse Before Iteration
```python
# GOOD: Seluruh partner dibaca dalam 1 query batch tunggal
partners = self.env["res.partner"].browse(partner_ids)
for partner in partners:
    print(partner.name)   # 1 query for all records!
```

### Eliminating N+1 with `mapped()` and Prefetched Sets
```python
# BAD (N queries for orders, plus N queries for partners):
for order in orders:
    print(order.partner_id.country_id.name)

# GOOD: Read relational fields across the entire recordset at once
countries = orders.mapped("partner_id.country_id")
for country in countries:
    print(country.name)
```

---

## 4. Chunking for Mass Data Processing (Anti-OOM)

When iterating through 50,000+ records, loading everything into memory will trigger Linux Out-Of-Memory (`OOM Kill`). Use chunked pagination:

```python
from odoo import api, models


class BatchDataProcessor(models.AbstractModel):
    _name = "batch.data.processor"
    _description = "Safe Batch Data Processing"

    @api.model
    def process_large_dataset(self, batch_size: int = 1000):
        domain = [("is_processed", "=", False)]
        offset = 0

        while True:
            # Query in discrete batches
            records = self.env["heavy.data.record"].search(
                domain,
                offset=offset,
                limit=batch_size,
                order="id asc"
            )
            if not records:
                break

            for record in records:
                record._execute_calculation()

            # Flush pending changes to PostgreSQL and free ORM memory
            self.env.flush_all()
            self.env.invalidate_all()

            offset += batch_size
```

---

## 5. Summary Checklist

- [x] **Values Isolation**: Every `create()` on related models is fed by a dedicated `_prepare_<model>_vals()` method.
- [x] **Lifecycle Separation**: Complex actions are divided into `_before_*`, `_action_*`, and `_after_*` hooks.
- [x] **Batch Browsing**: Never call `browse()` inside a loop; always browse the ID list first.
- [x] **Chunking Large Datasets**: Use `limit` and `offset` with `flush_all()` and `invalidate_all()` when processing massive data volumes.

