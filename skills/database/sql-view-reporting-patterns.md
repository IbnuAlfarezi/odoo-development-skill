# SQL View Reporting Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  SQL VIEW REPORTING PATTERNS                                                 ║
║  Analytical models, database views (_auto=False), and fast BI queries        ║
║  Use for Pivot, Graph, and List reports across massive datasets              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

In Odoo, computing complex analytics or aggregation through Python loops causes extreme latency and N+1 queries. The standard approach for analytical reporting is **SQL Views** (`_auto = False`).

A SQL View model is a read-only Odoo model backed by a PostgreSQL database `VIEW` instead of a physical table.

---

## 1. Core Model Rules (`_auto = False`)

1. **`_auto = False`**: Tells Odoo not to generate a physical PostgreSQL table.
2. **`readonly=True`**: All fields in a SQL View model **must** be marked as `readonly=True`.
3. **`_rec_name`**: Must be explicitly declared if the view lacks a standard `name` field.
4. **Primary Key `id`**: The SQL query **must** produce a unique integer column named `id` (e.g. `row_number() OVER () AS id` or using the primary key of the core table).

---

## 2. Standard Implementation (Modular `init()` Pattern)

Splitting the SQL query into modular methods (`_select()`, `_from()`, `_where()`, `_group_by()`) is the official OCA and core Odoo standard. This allows other custom modules to inherit and extend the view cleanly.

```python
from odoo import fields, models, tools


class SaleAnalysisReport(models.Model):
    _name = "sale.analysis.report"
    _description = "Sales Analysis BI Report"
    _auto = False
    _rec_name = "partner_id"
    _order = "date desc"

    date = fields.Date(string="Order Date", readonly=True)
    partner_id = fields.Many2one("res.partner", string="Customer", readonly=True)
    user_id = fields.Many2one("res.users", string="Salesperson", readonly=True)
    company_id = fields.Many2one("res.company", string="Company", readonly=True)
    product_id = fields.Many2one("product.product", string="Product", readonly=True)
    categ_id = fields.Many2one("product.category", string="Product Category", readonly=True)
    qty_ordered = fields.Float(string="Qty Ordered", readonly=True)
    qty_delivered = fields.Float(string="Qty Delivered", readonly=True)
    price_total = fields.Monetary(string="Total Amount", currency_field="currency_id", readonly=True)
    currency_id = fields.Many2one("res.currency", string="Currency", readonly=True)

    def _select(self) -> str:
        return """
            SELECT
                min(sol.id) AS id,
                so.date_order::date AS date,
                so.partner_id AS partner_id,
                so.user_id AS user_id,
                so.company_id AS company_id,
                so.currency_id AS currency_id,
                sol.product_id AS product_id,
                pt.categ_id AS categ_id,
                sum(sol.product_uom_qty) AS qty_ordered,
                sum(sol.qty_delivered) AS qty_delivered,
                sum(sol.price_total) AS price_total
        """

    def _from(self) -> str:
        return """
            FROM sale_order_line sol
            JOIN sale_order so ON (sol.order_id = so.id)
            JOIN product_product pp ON (sol.product_id = pp.id)
            JOIN product_template pt ON (pp.product_tmpl_id = pt.id)
        """

    def _where(self) -> str:
        return """
            WHERE so.state NOT IN ('draft', 'cancel')
        """

    def _group_by(self) -> str:
        return """
            GROUP BY
                so.date_order::date,
                so.partner_id,
                so.user_id,
                so.company_id,
                so.currency_id,
                sol.product_id,
                pt.categ_id
        """

    def init(self):
        tools.drop_view_if_exists(self.env.cr, self._table)
        self.env.cr.execute(
            f"""
            CREATE OR REPLACE VIEW {self._table} AS (
                {self._select()}
                {self._from()}
                {self._where()}
                {self._group_by()}
            )
            """
        )
```

---

## 3. Extending an Existing SQL View (`_inherit`)

When extending a SQL View from another module, override `_select()`, `_from()`, or `_group_by()` and trigger `init()`:

```python
class SaleAnalysisReport(models.Model):
    _inherit = "sale.analysis.report"

    warehouse_id = fields.Many2one("stock.warehouse", string="Warehouse", readonly=True)

    def _select(self) -> str:
        select_str = super()._select()
        return select_str + ", so.warehouse_id AS warehouse_id"

    def _group_by(self) -> str:
        groupby_str = super()._group_by()
        return groupby_str + ", so.warehouse_id"
```

---

## 4. Modern Odoo 18 Approach (`property _table_query`)

In modern Odoo 18, SQL views can also be defined using the property `_table_query` with the new safe `SQL()` query builder:

```python
from odoo import fields, models
from odoo.tools import SQL


class InventoryValuationReport(models.Model):
    _name = "inventory.valuation.report"
    _description = "Inventory Valuation Analysis"
    _auto = False
    _rec_name = "product_id"

    product_id = fields.Many2one("product.product", string="Product", readonly=True)
    location_id = fields.Many2one("stock.location", string="Location", readonly=True)
    quantity = fields.Float(string="On Hand Qty", readonly=True)

    @property
    def _table_query(self) -> SQL:
        return SQL(
            """
            SELECT
                min(sq.id) AS id,
                sq.product_id AS product_id,
                sq.location_id AS location_id,
                sum(sq.quantity) AS quantity
            FROM stock_quant sq
            GROUP BY sq.product_id, sq.location_id
            """
        )
```

---

## 5. XML Views for SQL Reports

SQL views are designed to be consumed by **Pivot** and **Graph** views in Odoo:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Pivot View -->
    <record id="view_sale_analysis_report_pivot" model="ir.ui.view">
        <field name="name">sale.analysis.report.pivot</field>
        <field name="model">sale.analysis.report</field>
        <field name="arch" type="xml">
            <pivot string="Sales Analysis" sample="1">
                <field name="categ_id" type="row"/>
                <field name="date" interval="month" type="col"/>
                <field name="price_total" type="measure"/>
                <field name="qty_ordered" type="measure"/>
            </pivot>
        </field>
    </record>

    <!-- Graph View -->
    <record id="view_sale_analysis_report_graph" model="ir.ui.view">
        <field name="name">sale.analysis.report.graph</field>
        <field name="model">sale.analysis.report</field>
        <field name="arch" type="xml">
            <graph string="Sales Trend" type="line" sample="1">
                <field name="date" interval="month"/>
                <field name="price_total" type="measure"/>
            </graph>
        </field>
    </record>

    <!-- Action -->
    <record id="action_sale_analysis_report" model="ir.actions.act_window">
        <field name="name">Sales Analysis</field>
        <field name="res_model">sale.analysis.report</field>
        <field name="view_mode">pivot,graph,list</field>
        <field name="context">{'search_default_current_year': 1}</field>
    </record>
</odoo>
```

---

## 6. Common Pitfalls & Anti-Patterns

| Anti-Pattern | Why it Fails | Correct Solution |
| :--- | :--- | :--- |
| Forgetting `readonly=True` on fields | Users or ORM may attempt to write/update, causing a PostgreSQL error (`cannot update view`). | Mark **all** fields as `readonly=True`. |
| Missing or non-unique `id` column | Odoo ORM crashes when rendering list or form views because it cannot identify unique records. | Use `min(table.id) AS id` or `row_number() OVER () AS id`. |
| Dropping views without `tools.drop_view_if_exists` | Module upgrade fails with `relation already exists` if the view schema changed. | Always call `tools.drop_view_if_exists(self.env.cr, self._table)` before `CREATE OR REPLACE VIEW`. |
| Using raw Python string concatenation with user input | Vulnerable to SQL Injection. | Use static query strings inside `_select()`, or Odoo 18's `SQL()` builder. |

