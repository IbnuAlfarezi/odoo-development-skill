# Manufacturing & MRP Architecture Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  MANUFACTURING & MRP PATTERNS                                                ║
║  Bill of Materials (BOM), Production Orders, Stock Move reservations,        ║
║  Work Centers, Work Orders, and Scrap Management                             ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo Manufacturing (MRP) orchestrates the conversion of raw material components into finished goods. The core architecture integrates Bill of Materials (`mrp.bom`), Manufacturing Orders (`mrp.production`), Work Centers (`mrp.workcenter`), and Work Orders (`mrp.workorder`) directly with the Inventory (`stock.move`) and Accounting valuation engines.

```
[ mrp.bom ] ──(Explodes)──▶ [ mrp.production ] ──▶ [ move_raw_ids (Components) ]
                                    │          ──▶ [ move_finished_ids (Finished) ]
                                    ▼
                             [ mrp.workorder ] ──▶ [ mrp.workcenter (Time & Cost) ]
```

---

## 1. Bill of Materials (`mrp.bom` & `mrp.bom.line`)

A Bill of Materials defines the recipe or assembly structure required to manufacture a product.

### BoM Types:
- **`normal` (Manufacture this product)**: Creates a physical manufacturing order (`mrp.production`).
- **`phantom` (Kit)**: No manufacturing order is created. When sold on a Sales Order or transferred in Inventory, the kit is automatically exploded into its individual raw components.

### Programmatic BoM Creation & Extension

```python
from odoo import api, fields, models


class MrpBom(models.Model):
    _inherit = 'mrp.bom'

    engineering_revision = fields.Char(string="Engineering Rev", default="REV-1.0", tracking=True)
    quality_inspection_required = fields.Boolean(string="Requires QA Signoff", default=False)

    @api.model
    def create_kit_bom(self, parent_product, components_data):
        """Helper method to programmatically create a Kit (Phantom) BoM.

        Args:
            parent_product: product.template or product.product record
            components_data: List of tuples [(product_id, qty), ...]
        """
        bom_vals = {
            'product_tmpl_id': parent_product.product_tmpl_id.id if hasattr(parent_product, 'product_tmpl_id') else parent_product.id,
            'product_qty': 1.0,
            'type': 'phantom',  # Kit
            'bom_line_ids': [
                (0, 0, {
                    'product_id': comp_id,
                    'product_qty': qty,
                })
                for comp_id, qty in components_data
            ],
        }
        return self.create(bom_vals)
```

---

## 2. Manufacturing Order Lifecycle (`mrp.production`)

A Manufacturing Order tracks the progress of assembly from raw material reservation to completion.

### Status Transitions:
```
[ draft ] ──(action_confirm)──▶ [ confirmed ] ──(action_assign)──▶ [ progress ] ──(button_mark_done)──▶ [ done ]
```

### Extending `mrp.production` with Custom Validation & Auto-Consumptions

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError


class MrpProduction(models.Model):
    _inherit = 'mrp.production'

    batch_tracking_code = fields.Char(string="Production Batch Code", copy=False, index=True)
    supervisor_id = fields.Many2one('res.users', string="Shift Supervisor", tracking=True)

    def action_confirm(self):
        """Confirm order, explode BoM into stock moves, and assign batch code."""
        for production in self:
            if not production.batch_tracking_code:
                # Generate unique production batch sequence
                production.batch_tracking_code = self.env['ir.sequence'].next_by_code('mrp.production.batch') or 'BATCH-GEN'
        return super().action_confirm()

    def button_mark_done(self):
        """Validate production completion and inventory valuation."""
        for production in self:
            # Enforce QA check if specified on BoM
            if production.bom_id.quality_inspection_required and not production.supervisor_id:
                raise UserError(_(
                    "Manufacturing Order %s requires shift supervisor signoff before marking as Done."
                ) % production.name)

            # Check that raw components have not been under-consumed beyond tolerance
            for move in production.move_raw_ids:
                if move.quantity == 0 and move.product_uom_qty > 0:
                    raise UserError(_(
                        "Component '%s' has 0 consumed quantity. Verify component consumption."
                    ) % move.product_id.display_name)

        return super().button_mark_done()
```

---

## 3. Stock Moves Linkages in Manufacturing

An `mrp.production` record links two sets of stock moves:
1. **`move_raw_ids`**: The components to be consumed.
   - Source Location: Warehouse Stock (`WH/Stock`)
   - Destination Location: Virtual Production Location (`Virtual Locations/Production`)
2. **`move_finished_ids`**: The completed finished goods (and by-products).
   - Source Location: Virtual Production Location (`Virtual Locations/Production`)
   - Destination Location: Warehouse Stock (`WH/Stock`)

### Programmatic Component Substitution

```python
from odoo import api, models


class MrpProduction(models.Model):
    _inherit = 'mrp.production'

    def action_substitute_raw_component(self, old_product, new_product, ratio=1.0):
        """Substitute a raw component on an active production order.

        Args:
            old_product: product.product to replace
            new_product: product.product substitute
            ratio: float conversion factor for quantity
        """
        self.ensure_one()
        target_moves = self.move_raw_ids.filtered(lambda m: m.product_id == old_product and m.state in ['draft', 'confirmed', 'waiting'])
        for move in target_moves:
            needed_qty = move.product_uom_qty * ratio
            move.write({'product_id': new_product.id, 'product_uom_qty': needed_qty})
            # Re-check stock reservation for new product
            move._action_assign()
```

---

## 4. Work Centers & Work Orders (`mrp.workcenter` & `mrp.workorder`)

Work centers represent machines, assembly stations, or human work teams with costs and capacities. Work orders represent individual operation steps.

### Work Order Execution & Time Tracking (`mrp.workorder`)

```python
from odoo import api, fields, models


class MrpWorkorder(models.Model):
    _inherit = 'mrp.workorder'

    def action_start_with_operator(self, employee):
        """Start work order timer linked to a specific operator."""
        self.ensure_one()
        self.button_start()
        # Tag the created productivity time log with employee
        time_log = self.time_ids.filtered(lambda l: not l.date_end)[:1]
        if time_log and hasattr(time_log, 'employee_id'):
            time_log.employee_id = employee.id
```

---

## 5. Scrap Management During Production (`stock.scrap`)

When raw materials are damaged during assembly, they must be scrapped from the active production order:

```python
from odoo import api, models


class MrpProduction(models.Model):
    _inherit = 'mrp.production'

    def action_scrap_raw_material(self, product, quantity, scrap_reason_code):
        """Create a scrap record directly linked to this manufacturing order."""
        self.ensure_one()
        scrap_location = self.env['stock.location'].search([
            ('scrap_location', '=', True),
            ('company_id', 'in', [self.company_id.id, False]),
        ], limit=1)

        scrap = self.env['stock.scrap'].create({
            'product_id': product.id,
            'scrap_qty': quantity,
            'product_uom_id': product.uom_id.id,
            'production_id': self.id,
            'location_id': self.location_src_id.id,
            'scrap_location_id': scrap_location.id,
            'origin': self.name,
        })
        scrap.action_validate()
        return scrap
```

---

## 6. Version Compatibility & Architecture Differences

| Feature | Odoo 16.0 | Odoo 17.0 & 18.0 | Odoo 19.0 & 20.0 |
| :--- | :--- | :--- | :--- |
| **Consumed Quantity** | `move.quantity_done` | `move.quantity` (ORM unified) | `move.quantity` |
| **BoM Line Constraints** | `_sql_constraints` | `_sql_constraints` | `models.Constraint` |
| **Work Order UI** | Standard Form | Shop Floor tablet view (`mrp_workorder`) | Shop Floor OWL 3 |
| **Access Rights** | `ir.model.access.csv` | `ir.model.access.csv` | `security/ir.access.csv` (v20) |

---

## 7. Best Practices & OCA Checklist for MRP

1. **Avoid Hardcoded Virtual Locations**: Never hardcode location IDs like `14` or `'Virtual Locations/Production'`. Always fetch via `self.env.ref('stock.stock_location_stock')` or `production.location_src_id`.
2. **Never Bypass `_action_assign()`**: When confirming production, allow the ORM stock reservation engine to create quants and pack operations naturally.
3. **Respect Unit of Measure (UoM)**: Always verify component conversions between purchase UoM and production BoM UoM (`product.uom_id._compute_quantity()`).

