# Version-Specific Testing & Modern Hoot Framework Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  VERSION-SPECIFIC TESTING & MODERN HOOT FRAMEWORK PATTERNS                   ║
║  Cross-version test matrix (v14-v20), Modern OWL testing with @odoo/hoot,    ║
║  Database Query Count Assertions (assertQueryCount), and Profiling           ║
║  Covers Odoo 14.0, 15.0, 16.0, 17.0, 18.0, 19.0, and 20.0                    ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Testing in Odoo has undergone massive architectural shifts across versions:
- In **Odoo 14–17**, frontend JavaScript testing used **QUnit**.
- In **Odoo 18**, Odoo introduced **Hoot** (`@odoo/hoot`) as the official modern test runner, deprecating QUnit.
- In **Odoo 19 & 20**, QUnit is **completely removed**, `SavepointCase` is replaced by `TransactionCase`, `groups_id` is renamed to `group_ids`, model constraints use `models.Constraint`, and security testing uses unified `ir.access`.

---

## 1. Cross-Version Testing Matrix

| Testing Feature | Odoo 14 – 15 | Odoo 16 – 17 | Odoo 18 | Odoo 19 | Odoo 20 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Frontend Framework** | `QUnit.test()` | `QUnit.test()` | **Hoot (`@odoo/hoot`)**, QUnit deprecated | **Hoot (`@odoo/hoot`)**, QUnit deprecated | **Hoot Only**, QUnit **REMOVED** |
| **Component Mounting** | `mount(Component)` | `mount(Component)` | `await mountWithCleanup(Component)` | `await mountWithCleanup(Component)` | `await mountWithCleanup(Component)` |
| **Base Test Class** | `SavepointCase` & `TransactionCase` | `SavepointCase` **REMOVED** | `TransactionCase` | `TransactionCase` | `TransactionCase` |
| **Record Creation** | Tuples `(0, 0, {...})` | `Command` class | `Command` class | `Command` class | `Command` class |
| **Security Assertions**| `check_access_rights()` | `check_access_rights()` | `check_access()` (old deprecated) | `check_access()` (old deprecated) | `ir.access` (`security/ir.access.csv`), old **REMOVED** |
| **Constraint Errors** | `IntegrityError` (`_sql_constraints`) | `IntegrityError` | `IntegrityError` | `ValidationError` (`models.Constraint`) | `ValidationError` (`models.Constraint`) |
| **User Groups Field** | `'groups_id'` | `'groups_id'` | `'groups_id'` | **`'group_ids'`** (Field renamed) | **`'group_ids'`** |
| **Query Assertions** | Manual inspection | `self.assertQueryCount()` | `self.profile()` + `assertQueryCount()` | `self.profile()` + `@warmup` | `self.profile()` + `@warmup` |

---

## 2. Modern Frontend Testing with `@odoo/hoot` (Odoo 18, 19, 20)

`@odoo/hoot` is Odoo's fast, modern, Jest/Vitest-like test runner designed specifically for OWL components.

### A. Manifest Registration (`__manifest__.py`)
Frontend tests in Odoo 18+ must be declared in the `web.unit_tests` bundle:

```python
{
    'name': 'My Module',
    'version': '18.0.1.0.0',
    'depends': ['web'],
    'assets': {
        'web.unit_tests': [
            'my_module/static/tests/**/*.js',
        ],
    },
    'license': 'LGPL-3',
}
```

### B. Testing an OWL Component (`static/tests/my_component_tests.js`)

```javascript
import { describe, expect, test } from "@odoo/hoot";
import { click, edit } from "@odoo/hoot-dom";
import { mountWithCleanup } from "@web/../web_helpers";
import { MyCounterComponent } from "@my_module/components/counter";

describe("MyCounterComponent Unit Tests", () => {
    test("renders initial count and increments on button click", async () => {
        // 1. Mount component in isolated virtual DOM
        const component = await mountWithCleanup(MyCounterComponent, {
            props: {
                initialValue: 5,
            },
        });

        // 2. Assert initial rendered text
        expect(".counter-value").toHaveText("Count: 5");

        // 3. Simulate user click using hoot-dom
        await click("button.btn-increment");

        // 4. Verify reactive DOM update
        expect(".counter-value").toHaveText("Count: 6");
        expect(component.state.count).toBe(6);
    });

    test("handles input field changes", async () => {
        await mountWithCleanup(MyCounterComponent);

        await edit("input.counter-input", "42");
        expect("input.counter-input").toHaveValue("42");
    });
});
```

### C. Migration Guide: QUnit (v14-17) vs Hoot (v18-20)

| Concept | Legacy QUnit (v14–v17) | Modern Hoot (v18–v20) |
| :--- | :--- | :--- |
| **Suite Declaration** | `QUnit.module("Suite Name", (hooks) => {...})` | `describe("Suite Name", () => {...})` |
| **Test Case** | `QUnit.test("test name", async (assert) => {...})` | `test("test name", async () => {...})` |
| **Equality Check** | `assert.strictEqual(a, b)` | `expect(a).toBe(b)` |
| **Deep Equality** | `assert.deepEqual(objA, objB)` | `expect(objA).toEqual(objB)` |
| **DOM Checks** | `assert.containsOnce(target, ".my-class")` | `expect(".my-class").toHaveCount(1)` |
| **DOM Text** | `assert.strictEqual(target.querySelector("span").innerText, "OK")`| `expect("span").toHaveText("OK")` |
| **DOM Click** | `await click(target.querySelector("button"))` | `await click("button")` |
| **Component Mount** | `await mount(MyComponent, { target })` | `await mountWithCleanup(MyComponent, { props })` |

---

## 3. Database Query Count & Profiling Testing

In high-concurrency enterprise modules (OCA standard), tests must ensure that batch operations do not trigger N+1 SQL queries.

### A. Testing Query Counts with `self.assertQueryCount()`

```python
from odoo.tests import TransactionCase, tagged


@tagged('post_install', '-at_install')
class TestOrderPerformance(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Seed 50 orders
        cls.partners = cls.env['res.partner'].create([
            {'name': f'Partner {i}'} for i in range(50)
        ])
        cls.orders = cls.env['sale.order'].create([
            {'partner_id': p.id} for p in cls.partners
        ])

    def test_batch_confirm_query_count(self):
        """Ensure batch confirming 50 orders executes a fixed number of queries (O(1), not O(N))"""
        # Warm the cache first to isolate the business logic queries
        self.orders.mapped('order_line')

        # Assert that confirming all 50 orders takes exactly or under 15 queries
        with self.assertQueryCount(__system__=14):
            self.orders.action_confirm()
```

### B. Profiling Test Execution (`with self.profile():`)

Use Odoo's built-in profiler inside a test method to capture slow queries, memory allocations, and method call stacks:

```python
    def test_profile_invoice_generation(self):
        """Profile invoice generation to analyze bottlenecks"""
        with self.profile(description="Invoice Batch Creation"):
            with self.assertQueryCount(__system__=8):
                self.orders._create_invoices()
```

---

## 4. Cross-Version Backend Test Case Blueprint

Use this version-adaptive test setup to write tests that run reliably across different versions:

```python
from odoo.tests import TransactionCase, tagged
from odoo.fields import Command
from odoo.exceptions import AccessError, ValidationError


@tagged('post_install', '-at_install')
class TestVersionAdaptive(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Handle res.users group field rename:
        # 'groups_id' in v14-v18 -> 'group_ids' in v19-v20
        group_field = 'group_ids' if 'group_ids' in cls.env['res.users']._fields else 'groups_id'

        cls.restricted_user = cls.env['res.users'].create({
            'name': 'Restricted Salesman',
            'login': 'restricted_salesman',
            'email': 'restricted@example.com',
            group_field: [Command.set([
                cls.env.ref('base.group_user').id,
            ])],
        })

    def test_access_permissions_by_version(self):
        """Test access permissions compatible with v16-v18 and v20 ir.access"""
        record = self.env['my.model'].create({'name': 'Confidential'})
        user_env_record = record.with_user(self.restricted_user)

        # In Odoo 18+, use check_access() instead of check_access_rights / check_access_rule
        if hasattr(user_env_record, 'check_access'):
            self.assertTrue(user_env_record.has_access('read'))
            with self.assertRaises(AccessError):
                user_env_record.check_access('unlink')
        else:
            # Fallback for v14-v17
            with self.assertRaises(AccessError):
                user_env_record.unlink()
```

---

## 5. Running Tests by Version via CLI

```bash
# 1. Run all tests for a specific module
odoo-bin -c odoo.conf -d test_db --test-enable --stop-after-init -i my_module --test-tags /my_module

# 2. Run only Unit tests (skip tours)
odoo-bin -c odoo.conf -d test_db --test-enable --stop-after-init --test-tags /my_module:TestMyModel

# 3. Run Hoot frontend tests via headless Chrome
odoo-bin -c odoo.conf -d test_db --test-enable --stop-after-init --test-tags /my_module.hoot
```

