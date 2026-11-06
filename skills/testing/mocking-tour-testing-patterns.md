# Mocking & Browser Tour Testing Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  MOCKING & BROWSER TOUR TESTING PATTERNS                                     ║
║  Integration testing with HttpCase, web tours, and external API / time mocks ║
║  Ensures seamless Python-JavaScript communication and CI/CD reliability      ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Testing Odoo applications requires more than isolated Python unit tests:
1. **End-to-End Tour Testing (`HttpCase`)**: Verifies that the OWL frontend and Python backend communicate properly during user navigation.
2. **External API Mocking (`unittest.mock`)**: Tests external integrations (payment gateways, courier APIs, webhooks) without making real network calls.
3. **Time Mocking (`freezegun`)**: Tests expiration dates, subscription renewals, and cron timelines without waiting.

---

## 1. Browser Tour Testing (`HttpCase` & `start_tour`)

A tour consists of two components:
- A **JavaScript step sequence** defining what to click and fill in the browser.
- A **Python test case** inheriting from `HttpCase` that launches a headless Chromium instance to execute the tour.

### Step 1: Define Tour Steps in JavaScript

Place in `static/src/tests/tours/sale_order_tour.js`:

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { stepUtils } from "@web_tour/tour_service/tour_utils";

registry.category("web_tour.tours").add("mining_equipment_tour", {
    url: "/web",
    test: true,
    steps: () => [
        // 1. Open the app from the main app switcher
        stepUtils.showAppsMenuItem(),
        {
            content: "Open Mining App",
            trigger: '.o_app[data-menu-xmlid="mining_core.menu_mining_root"]',
            run: "click",
        },
        // 2. Click Create button in list view
        {
            content: "Click New Equipment",
            trigger: ".o_list_button_add",
            run: "click",
        },
        // 3. Fill the form field
        {
            content: "Enter Equipment Name",
            trigger: ".o_field_widget[name='name'] input",
            run: "fill Excavator CAT 320",
        },
        // 4. Save the form
        {
            content: "Save Equipment Form",
            trigger: ".o_form_button_save",
            run: "click",
        },
        // 5. Assert the record was saved
        {
            content: "Verify saved badge",
            trigger: ".o_field_widget[name='state']:contains('Active')",
            run: () => {}, // empty run verifies visibility
        },
    ],
});
```

### Step 2: Launch the Tour from Python Test

Place in `tests/test_tours.py`:

```python
from odoo.tests import HttpCase, tagged


@tagged("post_install", "-at_install")
class TestMiningTour(HttpCase):

    def test_mining_equipment_creation_tour(self):
        """Run headless browser tour as admin user."""
        self.start_tour(
            url_path="/web",
            tour_name="mining_equipment_tour",
            login="admin",
            timeout=120,
        )
```

> [!TIP]
> **Local Interactive Debugging:**  
> When developing tours locally, use `self.start_tour("/web", "my_tour", watch=True)` or `debug=True` to open a visible Chrome window with Developer Tools attached!

---

## 2. Mocking External HTTP APIs in Python Tests

Never let automated unit tests perform real network requests to external servers. Use `unittest.mock.patch` to intercept requests and simulate success/failure scenarios:

```python
from unittest.mock import MagicMock, patch
from odoo.tests.common import TransactionCase, tagged
from odoo.exceptions import UserError


@tagged("post_install", "-at_install")
class TestPaymentIntegration(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.order = cls.env["sale.order"].create({
            "partner_id": cls.env.ref("base.res_partner_12").id,
        })

    @patch("requests.post")
    def test_payment_charge_success(self, mock_post):
        """Simulate a successful external payment gateway response."""
        # 1. Configure the mock response
        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "status": "success",
            "transaction_id": "TXN_998877",
            "amount": 500.0,
        }
        mock_post.return_value = mock_response

        # 2. Call your module method that triggers requests.post
        payment_service = self.env["payment.integration.service"]
        res = payment_service.charge_order(self.order, amount=500.0)

        # 3. Assert external call parameters and internal state
        mock_post.assert_called_once()
        self.assertEqual(res["transaction_id"], "TXN_998877")
        self.assertEqual(self.order.payment_status, "paid")

    @patch("requests.post")
    def test_payment_gateway_timeout(self, mock_post):
        """Simulate gateway failure or timeout handling."""
        mock_post.side_effect = TimeoutError("Connection timed out.")

        payment_service = self.env["payment.integration.service"]
        with self.assertRaises(UserError):
            payment_service.charge_order(self.order, amount=500.0)

        self.assertEqual(self.order.payment_status, "failed")
```

---

## 3. Mocking Time & Deadlines (`freezegun`)

Test time-based logic (e.g. overdue invoice detection, expiring access tokens, subscription auto-renewals) without waiting for actual days to pass:

```python
from freezegun import freeze_time
from odoo.tests.common import TransactionCase, tagged


@tagged("post_install", "-at_install")
class TestEquipmentMaintenanceSchedule(TransactionCase):

    def test_preventive_maintenance_due(self):
        """Test overdue status when date advances past the scheduled maintenance."""
        # Set base date
        with freeze_time("2026-01-01 08:00:00"):
            equipment = self.env["mining.equipment"].create({
                "name": "Haul Truck #101",
                "next_service_date": "2026-01-15",
            })
            self.assertFalse(equipment.is_overdue)

        # Fast forward time past the service date
        with freeze_time("2026-01-16 09:00:00"):
            # Recompute or read overdue status
            self.assertTrue(equipment.is_overdue)
```

---

## 4. Testing CLI Command Reference

To run specific test classes or tours efficiently from the command line:

```bash
# Run all tests in a custom module
./odoo-bin -d test_db -u my_module --test-enable --stop-after-init

# Run only a specific test class
./odoo-bin -d test_db --test-enable --stop-after-init \
  --test-tags /my_module:TestMiningTour

# Run a specific test method
./odoo-bin -d test_db --test-enable --stop-after-init \
  --test-tags /my_module:TestPaymentIntegration.test_payment_charge_success
```

---

## 5. Summary Checklist

- [x] **Browser Tour Tests**: Use `HttpCase` and `start_tour()` to verify end-to-end UI actions.
- [x] **Mock External HTTP**: Always `@patch("requests.post")` or `requests.get` to prevent external network traffic during test suites.
- [x] **Freeze Time**: Use `freezegun.freeze_time()` for predictable testing of expirations and deadlines.
- [x] **Tag Tests Appropriately**: Use `@tagged("post_install", "-at_install")` so tests run with full dependency graph loaded.

