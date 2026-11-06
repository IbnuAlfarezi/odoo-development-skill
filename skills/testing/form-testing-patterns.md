# Form View Testing Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  FORM VIEW TESTING PATTERNS                                                  ║
║  Simulate user UI interaction, onchange triggers, and view validations       ║
║  Use odoo.tests.Form helper to test business logic as experienced in the UI  ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

When writing unit tests in Python, calling `model.create()` or writing to fields directly **does not trigger `@api.onchange` methods** or view-level default values. 

To test business logic exactly as a user experiences it in the web client, Odoo provides the **`odoo.tests.Form`** test helper.

---

## 1. Why `odoo.tests.Form` is Essential

| Feature | Direct ORM `create()` | Using `Form(record)` |
| :--- | :--- | :--- |
| **Executes `@api.onchange`** | ❌ No | ✅ Yes (automatically triggers on field change) |
| **Applies View Default Values** | ❌ No | ✅ Yes (loads view context defaults) |
| **Validates View `required` Fields** | ❌ No | ✅ Yes (raises error if missing) |
| **Tests One2many Sub-views** | ⚠️ Manual dictionary | ✅ Dedicated sub-form helper |

---

## 2. Basic Form Testing Pattern

```python
from odoo.tests.common import TransactionCase, tagged
from odoo.tests import Form


@tagged("post_install", "-at_install")
class TestSaleOrderForm(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env["res.partner"].create({
            "name": "Acme Corp",
            "property_payment_term_id": cls.env.ref("account.account_payment_term_immediate").id,
        })
        cls.product = cls.env["product.product"].create({
            "name": "Industrial Gear",
            "list_price": 250.0,
        })

    def test_sale_order_form_onchange(self):
        """Test that changing partner automatically sets payment terms via onchange."""
        # 1. Initialize Form on the model
        order_form = Form(self.env["sale.order"])

        # 2. Set fields as if typing in the UI
        order_form.partner_id = self.partner

        # 3. Assert onchange behavior was executed automatically
        self.assertEqual(
            order_form.payment_term_id,
            self.partner.property_payment_term_id,
            "Payment term should be automatically filled by onchange from customer."
        )

        # 4. Save the form (calls create() internally with verified view values)
        order = order_form.save()
        self.assertTrue(order.id)
        self.assertEqual(order.state, "draft")
```

---

## 3. Testing One2many Sub-forms

To test adding sub-lines (such as order lines or invoice lines) with their own onchange triggers, use `subform.new()`:

```python
    def test_sale_order_lines_onchange(self):
        """Test adding order lines and verifying price calculation."""
        order_form = Form(self.env["sale.order"])
        order_form.partner_id = self.partner

        # Open sub-form for order_line
        with order_form.order_line.new() as line:
            line.product_id = self.product
            # Triggered onchange should set product name and unit price
            self.assertEqual(line.price_unit, 250.0)
            line.product_uom_qty = 4.0

        # Save main form
        order = order_form.save()

        # Assert total computation
        self.assertEqual(len(order.order_line), 1)
        self.assertEqual(order.amount_untaxed, 1000.0)
```

---

## 4. Editing an Existing Record via Form

You can also pass an existing record to `Form` to test edit operations:

```python
    def test_edit_existing_order(self):
        order = self.env["sale.order"].create({
            "partner_id": self.partner.id,
        })

        # Load existing record into Form
        with Form(order) as form:
            form.client_order_ref = "PO-2026-001"
            with form.order_line.new() as line:
                line.product_id = self.product
                line.product_uom_qty = 2.0

        # After the context manager exits, changes are saved to the record
        self.assertEqual(order.client_order_ref, "PO-2026-001")
        self.assertEqual(order.amount_untaxed, 500.0)
```

---

## 5. Testing with Specific Form Views

If a model has multiple form views, you can specify the exact view XML ID to validate:

```python
    def test_custom_view_validation(self):
        order_form = Form(
            self.env["sale.order"],
            view="sale.view_order_form"
        )
        order_form.partner_id = self.partner
        order = order_form.save()
        self.assertTrue(order.id)
```

---

## 6. Testing Validation Errors in Form

Ensure that UI constraint checks fail when mandatory view fields are omitted:

```python
from odoo.exceptions import AssertionError as OdooAssertionError


    def test_missing_mandatory_field_raises(self):
        order_form = Form(self.env["sale.order"])
        # partner_id is required in the form view
        with self.assertRaises(AssertionError):
            order_form.save()
```

---

## 7. Best Practices Checklist

- ✅ **Use `Form` when testing `@api.onchange`** logic. Direct `create()` ignores onchange!
- ✅ **Test sub-records with `with form.one2many_field.new() as item:`** to replicate real user input flow.
- ✅ **Tag your test classes** with `@tagged('post_install', '-at_install')` for optimal test run order.
- ✅ **Clean database assertions:** `Form.save()` creates clean records within the current `TransactionCase` rollback boundary.

