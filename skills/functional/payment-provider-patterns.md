# Payment Provider & Gateway Integration Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  PAYMENT PROVIDER & GATEWAY INTEGRATION PATTERNS                             ║
║  Integration with external payment gateways (Midtrans, Xendit, Stripe, etc.) ║
║  Provider credentials, transaction lifecycles, webhooks, and tokenization    ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo's `payment` module provides an abstraction layer for integrating external payment gateways (such as Stripe, PayPal, Adyen, Midtrans, Xendit, Razorpay). Creating a custom payment gateway requires extending three core models and implementing a secure webhook controller:

1. **`payment.provider`**: Manages gateway settings, credentials, merchant IDs, and operational modes (`test` vs `enabled`).
2. **`payment.transaction`**: Manages individual payment lifecycles (`draft` → `pending` → `authorized` → `done` / `cancel` / `error`), token payments, and refunds.
3. **`payment.token`**: Stores reusable customer payment tokens for 1-click checkout and subscriptions.
4. **Webhook Controller**: Receives asynchronous notifications from the gateway, validates cryptographic HMAC signatures, and triggers transaction state updates.

---

## 1. Module Manifest (`__manifest__.py`)

A payment provider module must depend on `payment` (and optionally `account` or `website_sale` if providing checkout templates):

```python
{
    'name': 'Payment Provider: MyGateway',
    'version': '18.0.1.0.0',
    'category': 'Accounting/Payment Providers',
    'sequence': 350,
    'summary': 'Integrate MyGateway Payment Service with Odoo',
    'depends': ['payment'],
    'data': [
        'views/payment_provider_views.xml',
        'views/payment_mygateway_templates.xml',
        'data/payment_provider_data.xml',
    ],
    'post_init_hook': 'post_init_hook',
    'uninstall_hook': 'uninstall_hook',
    'license': 'LGPL-3',
}
```

---

## 2. Payment Provider Model (`models/payment_provider.py`)

Extend `payment.provider` to register the unique provider code and configure credential fields:

```python
from odoo import fields, models


class PaymentProvider(models.Model):
    _inherit = 'payment.provider'

    code = fields.Selection(
        selection_add=[('mygateway', "MyGateway")],
        ondelete={'mygateway': 'set default'},
    )
    mygateway_merchant_id = fields.Char(
        string="Merchant ID",
        help="The Merchant ID provided by MyGateway.",
        required_if_provider='mygateway',
    )
    mygateway_server_key = fields.Char(
        string="Server Key / Secret",
        help="Secret key used to authenticate API requests and sign webhooks.",
        required_if_provider='mygateway',
        groups='base.group_system',  # Sensitive credential: restricted to settings admins
    )
    mygateway_client_key = fields.Char(
        string="Client Key",
        help="Public key used by frontend drop-in UI widgets.",
        required_if_provider='mygateway',
    )

    def _get_default_payment_method_codes(self):
        """Return the default payment method codes supported by this provider."""
        default_codes = super()._get_default_payment_method_codes()
        if self.code != 'mygateway':
            return default_codes
        return default_codes | {'card', 'bank_transfer', 'ewallet'}

    def _mygateway_get_api_url(self):
        """Return the appropriate API URL depending on provider state."""
        self.ensure_one()
        if self.state == 'enabled':
            return 'https://api.mygateway.com/v1'
        return 'https://api.sandbox.mygateway.com/v1'
```

---

## 3. Payment Transaction Model (`models/payment_transaction.py`)

The transaction model orchestrates payment processing, asynchronous status updates, tokenization, and refunds:

```python
import logging
import pprint
from odoo import _, api, models
from odoo.exceptions import ValidationError

_logger = logging.getLogger(__name__)


class PaymentTransaction(models.Model):
    _inherit = 'payment.transaction'

    def _get_specific_rendering_values(self, processing_values):
        """Pass rendering values to the redirection form or drop-in JS widget.

        Called when generating the payment form before redirection or popup initialization.
        """
        res = super()._get_specific_rendering_values(processing_values)
        if self.provider_code != 'mygateway':
            return res

        api_url = self.provider_id._mygateway_get_api_url()
        rendering_values = {
            'api_url': f"{api_url}/checkout",
            'merchant_id': self.provider_id.mygateway_merchant_id,
            'client_key': self.provider_id.mygateway_client_key,
            'reference': self.reference,
            'amount': self.amount,
            'currency': self.currency_id.name,
            'customer_email': self.partner_email,
        }
        return rendering_values

    def _send_payment_request(self):
        """Request the provider to charge a saved payment token (recurring/1-click)."""
        super()._send_payment_request()
        if self.provider_code != 'mygateway':
            return

        if not self.token_id:
            raise ValidationError(_("MyGateway: Cannot charge without a valid payment token."))

        # Make server-to-server HTTP request to charge token
        payload = {
            'merchant_id': self.provider_id.mygateway_merchant_id,
            'token_id': self.token_id.provider_ref,
            'reference': self.reference,
            'amount': self.amount,
            'currency': self.currency_id.name,
        }
        # Execute charge and process response
        response_data = self._mygateway_make_api_call('/charge-token', payload)
        self._process('mygateway', response_data)

    def _extract_reference(self, provider_code, payment_data):
        """Extract the unique transaction reference from webhook/redirect data."""
        if provider_code != 'mygateway':
            return super()._extract_reference(provider_code, payment_data)
        return payment_data.get('order_id') or payment_data.get('reference')

    def _apply_updates(self, payment_data):
        """Apply transaction status updates based on provider notification data.

        This method is invoked internally by `_process()`.
        """
        super()._apply_updates(payment_data)
        if self.provider_code != 'mygateway':
            return

        status = payment_data.get('transaction_status')
        gateway_tx_id = payment_data.get('transaction_id')
        self.provider_reference = gateway_tx_id

        if status in ['capture', 'settlement', 'success']:
            # Successful payment
            self._set_done()
            # If customer requested tokenization, generate token record
            if payment_data.get('saved_token_id') and not self.token_id:
                self._mygateway_tokenize_from_data(payment_data)
        elif status in ['pending', 'authorize']:
            self._set_pending()
        elif status in ['deny', 'cancel', 'expire']:
            self._set_canceled()
        else:
            _logger.warning("MyGateway: Unknown transaction status %s received", status)
            self._set_error(_("MyGateway: Received unrecognized payment status %s", status))

    def _mygateway_tokenize_from_data(self, payment_data):
        """Create a payment.token record for 1-click checkout."""
        token = self.env['payment.token'].create({
            'provider_id': self.provider_id.id,
            'partner_id': self.partner_id.id,
            'provider_ref': payment_data['saved_token_id'],
            'payment_details': payment_data.get('masked_card', 'Saved Card'),
            'verified': True,
        })
        self.token_id = token
        _logger.info("Created payment token %s for partner %s", token.id, self.partner_id.name)

    def _send_refund_request(self, amount_to_refund=None):
        """Execute refund request on gateway for this transaction."""
        super()._send_refund_request(amount_to_refund=amount_to_refund)
        if self.provider_code != 'mygateway':
            return

        refund_tx = self.env['payment.transaction'].search([
            ('source_transaction_id', '=', self.id),
            ('operation', '=', 'refund'),
        ], limit=1)

        payload = {
            'transaction_id': self.provider_reference,
            'refund_amount': amount_to_refund or self.amount,
            'reason': refund_tx.reference if refund_tx else "Customer Refund",
        }
        response = self._mygateway_make_api_call('/refund', payload)
        if response.get('status') == 'success':
            refund_tx._set_done()
        else:
            refund_tx._set_error(_("Refund failed: %s", response.get('message')))
```

---

## 4. Secure Webhook Controller (`controllers/main.py`)

Payment gateways send asynchronous HTTP POST notifications when a transaction succeeds or fails. The webhook endpoint must be public (`auth='public'`), disable CSRF (`csrf=False`), and strictly verify signatures:

```python
import hashlib
import hmac
import json
import logging
from odoo import http
from odoo.exceptions import Forbidden
from odoo.http import request

_logger = logging.getLogger(__name__)


class MyGatewayController(http.Controller):
    _webhook_url = '/payment/mygateway/webhook'

    @http.route(_webhook_url, type='http', auth='public', methods=['POST'], csrf=False)
    def mygateway_webhook(self):
        """Handle incoming webhook notification from MyGateway."""
        payload = request.httprequest.data.decode('utf-8')
        try:
            notification_data = json.loads(payload)
        except json.JSONDecodeError:
            _logger.warning("MyGateway: Invalid JSON received in webhook.")
            raise Forbidden("Invalid payload")

        _logger.info("MyGateway webhook received:\n%s", notification_data)

        # 1. Extract transaction reference
        order_id = notification_data.get('order_id')
        if not order_id:
            raise Forbidden("Missing order reference")

        tx = request.env['payment.transaction'].sudo().search([
            ('reference', '=', order_id),
            ('provider_code', '=', 'mygateway'),
        ], limit=1)

        if not tx:
            _logger.warning("MyGateway: No transaction found for order %s", order_id)
            raise Forbidden("Transaction not found")

        # 2. Cryptographic signature verification
        server_key = tx.provider_id.mygateway_server_key
        received_signature = request.httprequest.headers.get('X-Signature') or notification_data.get('signature_key')
        status_code = notification_data.get('status_code', '')
        gross_amount = notification_data.get('gross_amount', '')

        # Standard gateway formula (e.g., SHA512(order_id + status_code + gross_amount + server_key))
        raw_hash_string = f"{order_id}{status_code}{gross_amount}{server_key}"
        expected_signature = hashlib.sha512(raw_hash_string.encode('utf-8')).hexdigest()

        if not hmac.compare_digest(received_signature or '', expected_signature):
            _logger.error("MyGateway: Webhook signature mismatch for order %s!", order_id)
            raise Forbidden("Invalid webhook signature")

        # 3. Process transaction in sudo mode (with secure verified context)
        tx._process('mygateway', notification_data)

        # 4. Acknowledge receipt to gateway (HTTP 200)
        return request.make_response(
            json.dumps({'status': 'OK'}),
            headers=[('Content-Type', 'application/json')],
        )
```

---

## 5. View Configuration (`views/payment_provider_views.xml`)

Add credential fields to the `payment.provider` form view conditionally when `code == 'mygateway'`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="payment_provider_form_mygateway" model="ir.ui.view">
        <field name="name">payment.provider.form.mygateway</field>
        <field name="model">payment.provider</field>
        <field name="inherit_id" ref="payment.payment_provider_form"/>
        <field name="arch" type="xml">
            <group name="provider_credentials" position="inside">
                <group invisible="code != 'mygateway'">
                    <field name="mygateway_merchant_id" required="code == 'mygateway' and state != 'disabled'"/>
                    <field name="mygateway_client_key" required="code == 'mygateway' and state != 'disabled'"/>
                    <field name="mygateway_server_key" password="True" required="code == 'mygateway' and state != 'disabled'"/>
                </group>
            </group>
        </field>
    </record>
</odoo>
```

---

## 6. Version Compatibility Notes

| Feature | Odoo 16.0 | Odoo 17.0 & 18.0 | Odoo 19.0 & 20.0 |
| :--- | :--- | :--- | :--- |
| **Provider Views** | `attrs="{'invisible': [('code', '!=', 'mygateway')]}"` | `invisible="code != 'mygateway'"` | `invisible="code != 'mygateway'"` |
| **Notification Processing** | `_handle_notification_data()` / `_process_notification_data()` | `_process(provider_code, data)` & `_apply_updates()` | `_process()` & `_apply_updates()` |
| **Transaction Reference** | `_process_reference()` | `_extract_reference(provider_code, data)` | `_extract_reference(provider_code, data)` |
| **Refunds** | `_send_refund()` | `_send_refund_request(amount_to_refund)` | `_send_refund_request(amount_to_refund)` |
| **Security** | `ir.model.access.csv` | `ir.model.access.csv` | `security/ir.access.csv` (v20) |

---

## 7. Best Practices & Security Checklist

1. **Never Log Sensitive Keys**: Do not print `server_key`, raw credit card numbers, or CVVs in logs. Use masked values or hash identifiers.
2. **Strict Signature Verification**: Always verify `HMAC-SHA256` or `SHA512` before updating transaction state. Reject requests without matching signatures with HTTP 403 `Forbidden`.
3. **Idempotency**: Webhook notifications may be redelivered. Calling `_set_done()` multiple times on an already confirmed transaction must be idempotent.
4. **Amount & Currency Validation**: Always check that `gross_amount` matches `tx.amount` and `currency` matches `tx.currency_id.name` to prevent currency-tampering exploits.

