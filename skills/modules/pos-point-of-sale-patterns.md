# Point of Sale (POS) Architecture Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  POINT OF SALE (POS) FULLSTACK PATTERNS                                      ║
║  Backend Session lifecycle, POS Orders, Accounting Journal generation,       ║
║  and Frontend OWL POS Components (ProductScreen buttons, Popups, and Store)  ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo Point of Sale operates as a hybrid architecture:
1. **Frontend (Cashier Client)**: A Single Page Application (SPA) built with OWL running in the web browser, supporting offline mode, local barcode scanning, and direct receipt printing.
2. **Backend (ERP Core)**: Manages cash drawer controls, inventory movements (`stock.picking`), accounting journal entries, customer loyalty points, and cash drawer balance reconciliation.

---

## 1. Backend: POS Session Lifecycle (`pos.session`)

A POS Session represents an active register shift. It enforces cash accountability:

```
[ Opening Control ] ───▶ [ Opened ] ───▶ [ Closing Control ] ───▶ [ Closed ]
  (Count Cash Float)     (Take Orders)   (Count Drawer & Cash)    (Post Journal Entries)
```

### Extending `pos.session` for Shift Handover & Validation

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError


class PosSession(models.Model):
    _inherit = 'pos.session'

    cashier_note = fields.Text(string="Shift Handover Notes")
    cash_difference_acknowledged = fields.Boolean(
        string="Difference Acknowledged",
        help="Flagged if cashier accepts cash discrepancies during closing.",
    )

    def action_pos_session_closing_control(self, balancing_account=False, amount_to_balance=0, bank_payment_method_diffs=None):
        """Override session closing to enforce discrepancy validation."""
        for session in self:
            # Enforce cashier acknowledgment if cash drawer difference exceeds tolerance
            if abs(session.cash_register_difference) > 5.0 and not session.cash_difference_acknowledged:
                raise UserError(_(
                    "Cash discrepancy detected (Difference: %s). "
                    "Supervisor must acknowledge and approve the difference before closing this shift."
                ) % session.cash_register_difference)

        return super().action_pos_session_closing_control(
            balancing_account=balancing_account,
            amount_to_balance=amount_to_balance,
            bank_payment_method_diffs=bank_payment_method_diffs,
        )
```

---

## 2. Backend: Creating & Extending POS Orders (`pos.order`)

Orders taken in the browser are sent as JSON payloads to the backend controller (`/pos/create_from_ui`) which executes `create_from_ui`:

```python
from odoo import api, fields, models


class PosOrder(models.Model):
    _inherit = 'pos.order'

    membership_card_number = fields.Char(string="Membership Card #", index=True)
    custom_discount_approved_by = fields.Many2one('res.users', string="Discount Approver")

    @api.model
    def _order_fields(self, ui_order):
        """Map JSON fields from POS frontend into ORM create dictionary."""
        order_fields = super()._order_fields(ui_order)
        order_fields.update({
            'membership_card_number': ui_order.get('membership_card_number'),
            'custom_discount_approved_by': ui_order.get('discount_approver_id'),
        })
        return order_fields

    def _export_for_ui(self, order):
        """Export additional backend fields back to POS browser client."""
        result = super()._export_for_ui(order)
        result['membership_card_number'] = order.membership_card_number
        return result
```

---

## 3. Frontend OWL POS: Asset Registration in Manifest

In Odoo 17 and 18, POS frontend assets must be bundled under `point_of_sale._assets_pos`:

```python
{
    'name': 'POS Custom Features',
    'version': '18.0.1.0.0',
    'depends': ['point_of_sale'],
    'data': [
        'views/pos_order_views.xml',
    ],
    'assets': {
        'point_of_sale._assets_pos': [
            'my_pos_module/static/src/xml/**/*.xml',
            'my_pos_module/static/src/js/**/*.js',
            'my_pos_module/static/src/scss/**/*.scss',
        ],
    },
    'license': 'LGPL-3',
}
```

---

## 4. Frontend OWL: Adding Custom Control Button to `ProductScreen`

To add a custom button to the main sales screen in POS:

### Step 1: Create the Component (`static/src/js/custom_discount_button.js`)

```javascript
/** @odoo-module **/

import { Component } from "@odoo/owl";
import { _t } from "@web/core/l10n/translation";
import { ProductScreen } from "@point_of_sale/app/screens/product_screen/product_screen";
import { usePos } from "@point_of_sale/app/store/pos_hook";
import { useService } from "@web/core/utils/hooks";

export class CustomDiscountButton extends Component {
    static template = "my_pos_module.CustomDiscountButton";

    setup() {
        this.pos = usePos();
        this.popup = useService("popup");
    }

    async onClick() {
        const currentOrder = this.pos.get_order();
        if (!currentOrder) {
            return;
        }

        const selectedLine = currentOrder.get_selected_orderline();
        if (!selectedLine) {
            this.popup.add("ConfirmPopup", {
                title: _t("No Line Selected"),
                body: _t("Please select a product line in the cart first."),
            });
            return;
        }

        // Display a number prompt or custom confirmation dialog
        const { confirmed, payload: discountStr } = await this.popup.add("NumberPopup", {
            title: _t("Apply VIP Discount (%)"),
            startingValue: 10,
        });

        if (confirmed) {
            const discount = parseFloat(discountStr);
            if (!isNaN(discount) && discount >= 0 && discount <= 100) {
                selectedLine.set_discount(discount);
            }
        }
    }
}

// Register the button directly on the ProductScreen control buttons bar
ProductScreen.addControlButton({
    component: CustomDiscountButton,
    position: ["after", "OrderlineCustomerNoteButton"],
});
```

### Step 2: Create the OWL Template (`static/src/xml/custom_discount_button.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
    <t t-name="my_pos_module.CustomDiscountButton">
        <button class="control-button btn btn-light rounded-0 fw-bolder d-flex align-items-center"
                t-on-click="onClick">
            <i class="fa fa-percent me-2 text-primary" role="img" aria-label="VIP Discount"/>
            <span>VIP Discount</span>
        </button>
    </t>
</templates>
```

---

## 5. Frontend OWL: Custom Popup Dialog Component

Creating a specialized popup dialog in the POS UI:

### Step 1: Component Definition (`static/src/js/membership_popup.js`)

```javascript
/** @odoo-module **/

import { AbstractAwaitablePopup } from "@point_of_sale/app/popup/abstract_awaitable_popup";
import { useState } from "@odoo/owl";
import { _t } from "@web/core/l10n/translation";

export class MembershipCardPopup extends AbstractAwaitablePopup {
    static template = "my_pos_module.MembershipCardPopup";
    static defaultProps = {
        confirmText: _t("Confirm"),
        cancelText: _t("Cancel"),
        title: _t("Scan Member Card"),
    };

    setup() {
        super.setup();
        this.state = useState({ cardNumber: "" });
    }

    getPayload() {
        return this.state.cardNumber;
    }
}
```

### Step 2: Popup XML Template (`static/src/xml/membership_popup.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
    <t t-name="my_pos_module.MembershipCardPopup">
        <div class="popup popup-membership">
            <div class="modal-header">
                <h4 class="modal-title"><t t-esc="props.title"/></h4>
            </div>
            <div class="modal-body">
                <input type="text"
                       class="form-control form-control-lg text-center"
                       placeholder="Scan barcode or enter ID..."
                       t-model="state.cardNumber"
                       t-ref="autofocus"/>
            </div>
            <div class="footer modal-footer">
                <button class="btn btn-secondary" t-on-click="cancel">
                    <t t-esc="props.cancelText"/>
                </button>
                <button class="btn btn-primary" t-on-click="confirm">
                    <t t-esc="props.confirmText"/>
                </button>
            </div>
        </div>
    </t>
</templates>
```

---

## 6. Receipt Customization (Adding Custom Fields to Printed Slip)

Extend the `OrderReceipt` template to display custom fields on thermal receipts:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates id="template" xml:space="preserve">
    <t t-inherit="point_of_sale.OrderReceipt" t-inherit-mode="extension">
        <xpath expr="//div[hasclass('pos-receipt-contact')]" position="after">
            <div t-if="props.data.membership_card_number" class="pos-receipt-center-align my-1 fw-bolder">
                Member ID: <t t-esc="props.data.membership_card_number"/>
            </div>
        </xpath>
    </t>
</templates>
```

---

## 7. Version Differences & Migration

| Feature | Odoo 16.0 | Odoo 17.0 & 18.0 | Odoo 20.0 |
| :--- | :--- | :--- | :--- |
| **Asset Bundle** | `point_of_sale.assets` | `point_of_sale._assets_pos` | `point_of_sale._assets_pos` |
| **POS Hook** | `usePos()` | `usePos()` from `@point_of_sale/app/store/pos_hook` | OWL 3 signals |
| **Button Registration** | `ProductScreen.addControlButton` | `ProductScreen.addControlButton` | Component injection |
| **Popup Class** | `AbstractAwaitablePopup` | `AbstractAwaitablePopup` from `@point_of_sale/app/popup/...` | Standard `@odoo/owl` dialogs |
| **Template Syntax** | `t-esc` | `t-esc` / `t-out` | `t-out` mandatory |

---

## 8. Best Practices Checklist for POS Development

1. **Keep Button Handlers Non-Blocking**: POS operates offline; never perform synchronous network requests without a timeout or fallback.
2. **Synchronize Custom Data**: If adding custom fields to products or partners, inherit `_loader_params_product_product()` or `_loader_params_res_partner()` on `pos.session` in Python to include them in the initial load payload.
3. **Respect Taxes & Decimals**: Never recalculate tax lines in JS using simple multiplication; always use the POS tax engine (`this.pos.get_order().get_tax_details()`).

