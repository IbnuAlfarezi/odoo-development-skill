# eCommerce & Delivery Carrier Integration Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ECOMMERCE & DELIVERY CARRIER PATTERNS                                       ║
║  Shopping Cart lifecycle (website_sale), Multi-step Checkout,                ║
║  Custom Shipping Carrier computation (delivery.carrier), and Tracking Labels ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo's online retail operations are powered by two primary modules:
1. **`website_sale`**: Manages the customer-facing online shop, digital product catalog, customer cart sessions, and the multi-step checkout funnel.
2. **`delivery`**: Provides an extensible framework for calculating shipping costs (`delivery.carrier`), integrating third-party courier APIs (FedEx, DHL, custom local logistics), and generating tracking numbers on warehouse pickings.

---

## 1. eCommerce Shopping Cart Lifecycle (`website_sale`)

In Odoo, an active eCommerce shopping cart is represented under the hood by a standard `sale.order` in `draft` state linked to the visitor's website session.

### A. Extending the Cart Engine (Python Controller & Model)

```python
from odoo import api, fields, models
from odoo.http import request


class SaleOrder(models.Model):
    _inherit = 'sale.order'

    delivery_instructions = fields.Text(string="Special Delivery Instructions")
    is_gift = fields.Boolean(string="Is a Gift", default=False)
    gift_message = fields.Char(string="Gift Message")

    def _cart_update_order_line(self, product_id, quantity, order_line_id=None, **kwargs):
        """Override cart line updates to inject custom eCommerce line values."""
        order_line = super()._cart_update_order_line(product_id, quantity, order_line_id=order_line_id, **kwargs)
        if kwargs.get('gift_message') and order_line:
            order_line.write({'name': f"{order_line.name}\n[Gift Message: {kwargs['gift_message']}]"})
        return order_line
```

### B. Accessing Cart Data via Website Controller

```python
from odoo import http
from odoo.addons.website_sale.controllers.main import WebsiteSale
from odoo.http import request


class CustomWebsiteSale(WebsiteSale):

    @http.route(['/shop/add_instructions'], type='json', auth='public', website=True)
    def update_delivery_instructions(self, instructions, is_gift=False, gift_message=""):
        """Save customer delivery instructions to the active cart."""
        order = request.website.sale_get_order()
        if not order:
            return {'status': 'error', 'message': 'No active cart found'}

        order.sudo().write({
            'delivery_instructions': instructions,
            'is_gift': is_gift,
            'gift_message': gift_message if is_gift else False,
        })
        return {'status': 'success'}
```

---

## 2. Delivery Carrier Integration (`delivery.carrier`)

Creating a custom shipping provider (e.g., custom courier service with dynamic API pricing or zone calculation) requires inheriting `delivery.carrier`.

### A. Carrier Model Extension (`models/delivery_carrier.py`)

```python
from odoo import _, fields, models
from odoo.exceptions import UserError


class DeliveryCarrier(models.Model):
    _inherit = 'delivery.carrier'

    delivery_type = fields.Selection(
        selection_add=[('custom_express', "Custom Express Courier")],
        ondelete={'custom_express': 'set default'},
    )
    custom_api_key = fields.Char(string="Courier API Key", groups="base.group_system")
    custom_api_secret = fields.Char(string="Courier Secret", groups="base.group_system")
    fixed_insurance_fee = fields.Float(string="Fixed Insurance Fee", default=5.0)

    def custom_express_rate_shipment(self, order):
        """Compute shipping cost for a sales order or cart.

        Returns a dictionary with:
        - success: boolean
        - price: calculated shipping cost in order currency
        - error_message: error string if failed
        - warning_message: optional warning string
        """
        self.ensure_one()
        # Calculate package weight
        total_weight = sum(line.product_id.weight * line.product_uom_qty for line in order.order_line if not line.is_delivery)

        # Free shipping threshold check (standard Odoo carrier feature)
        if self.free_over and order.amount_untaxed >= self.amount:
            return {
                'success': True,
                'price': 0.0,
                'error_message': False,
                'warning_message': _("Free Shipping threshold unlocked!"),
            }

        # Custom pricing logic (Base fee + per kg calculation)
        base_rate = 10.0
        weight_rate = max(1.0, total_weight) * 2.5
        total_price = base_rate + weight_rate + self.fixed_insurance_fee

        return {
            'success': True,
            'price': total_price,
            'error_message': False,
            'warning_message': False,
        }

    def custom_express_send_shipping(self, pickings):
        """Generate shipping labels and tracking numbers when transferring stock.picking."""
        res = []
        for picking in pickings:
            # Generate or fetch tracking number from courier API
            tracking_number = f"EXP-{picking.id}-{picking.name.replace('/', '')}"
            picking.carrier_tracking_ref = tracking_number

            res.append({
                'exact_price': picking.carrier_id.fixed_price or 15.0,
                'tracking_number': tracking_number,
            })
            picking.message_post(body=_(
                "Shipping label generated for Custom Express. Tracking: %s"
            ) % tracking_number)
        return res

    def custom_express_cancel_shipment(self, pickings):
        """Cancel shipment on courier system."""
        for picking in pickings:
            picking.carrier_tracking_ref = False
            picking.message_post(body=_("Custom Express shipment was cancelled."))
```

---

## 3. Shipping Options in eCommerce Checkout View

Extend the checkout delivery selection template to show delivery estimates:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <template id="custom_checkout_delivery" inherit_id="website_sale.checkout_delivery">
        <xpath expr="//div[hasclass('o_delivery_carrier_select')]" position="inside">
            <t t-if="delivery.delivery_type == 'custom_express'">
                <span class="badge text-bg-info ms-2">
                    <i class="fa fa-bolt"/> Same Day Available
                </span>
            </t>
        </xpath>
    </template>
</odoo>
```

---

## 4. Best Practices Checklist for eCommerce & Delivery

1. **Cart Isolation**: Always use `request.website.sale_get_order()` rather than searching `sale.order` by customer ID, to prevent session hijacking across different web visitors.
2. **Shipping Computation Return Format**: The rate calculation method (`<delivery_type>_rate_shipment`) MUST return a dictionary containing `'success'`, `'price'`, `'error_message'`, and `'warning_message'`.
3. **Product Weight Integrity**: Ensure all physical sellable products have valid `weight` and `volume` fields configured; otherwise, courier API requests will fail during rate calculation.

