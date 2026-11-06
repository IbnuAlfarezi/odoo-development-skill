# sudo() Security Audit & Access Protection Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  SUDO() SECURITY AUDIT & PRIVILEGE ESCALATION PATTERNS                       ║
║  Safe sudo() usage, IDOR vulnerability prevention, field-level access leaks, ║
║  and access rights validation across Odoo 16, 17, 18, and 20                 ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

The `sudo()` method is the most frequently misused API in custom Odoo development. Bypassing the ORM's security checks without proper validation introduces critical vulnerabilities:
1. **IDOR (Insecure Direct Object Reference)**: Users accessing, modifying, or deleting records of other companies or departments.
2. **Privilege Escalation**: Unauthorized users triggering manager-only business transitions.
3. **Field-Level Data Leaks**: Exposing confidential fields (costs, margins, salaries) to unauthorized users.

---

## 1. "Visibility != Security" (The Golden Rule)

> **Important**: Hiding a field in an XML view using `groups="..."` **does not protect it from unauthorized access**.

```xml
<!-- ❌ INSUFFICIENT SECURITY: Only hides from the web UI -->
<!-- Users can still read or write this field via JSON-RPC, external API, or Excel export! -->
<field name="cost_price" groups="account.group_account_manager"/>
```

### ✅ The Correct Pattern: Declare `groups` on the Python Model

To truly restrict read and write access at the ORM layer, declare `groups` on the Python field:

```python
from odoo import fields, models


class ProductTemplate(models.Model):
    _inherit = "product.template"

    # Protected at the ORM level — unauthorized users get an AccessError if they query it
    cost_price = fields.Float(
        string="Internal Cost Price",
        groups="account.group_account_manager",
    )
```

---

## 2. Preventing IDOR Vulnerabilities with `sudo()`

### ❌ The Vulnerable Pattern: Blind `sudo()` on User Inputs
```python
@http.route("/portal/my/document/<int:doc_id>/cancel", type="json", auth="user")
def cancel_document(self, doc_id, **kw):
    # FATAL VULNERABILITY: An attacker can pass any doc_id (even belonging to another user or company!)
    # and cancel their documents because sudo() strips all record rules and multi-company checks!
    doc = request.env["custom.document"].sudo().browse(doc_id)
    doc.action_cancel()
    return {"status": "cancelled"}
```

### ✅ The Secure Pattern: Validate Ownership Before Elevating Privileges

Always verify that the current user has access to the record **before** executing privileged operations:

```python
@http.route("/portal/my/document/<int:doc_id>/cancel", type="json", auth="user")
def cancel_document(self, doc_id, **kw):
    # 1. Browse WITHOUT sudo to enforce current user's record rules
    doc = request.env["custom.document"].browse(doc_id)

    # 2. Programmatically verify write access (raises AccessError if unauthorized)
    doc.check_access_rule("write")
    doc.check_access_rights("write")

    # 3. Elevate privilege only for the specific internal operation if necessary
    doc.sudo()._execute_system_cancellation()
    return {"status": "cancelled"}
```

---

## 3. Safe Alternatives to `sudo()`

Instead of switching to the omnipotent superuser, use targeted context and user switching:

### A. Switch to a Specific User with `with_user()`
```python
# Test or execute as an auditor or specific user
auditor_user = self.env.ref("my_module.user_auditor")
records = self.env["account.move"].with_user(auditor_user).search([])
```

### B. Narrow Scope with `with_context()`
```python
# If you only need to access multi-company data, switch company context safely:
records = self.env["sale.order"].with_context(
    allowed_company_ids=[company.id]
).search([])
```

---

## 4. Legitimate Use Cases for `sudo()`

When is `sudo()` genuinely acceptable?

| Use Case | Why `sudo()` is Legitimate | Precaution |
| :--- | :--- | :--- |
| **Sequence Number Generation** | Regular users need to increment sequence numbers stored in system tables (`ir.sequence`). | Only call `.sudo()` on `self.env['ir.sequence']`, never on the business record. |
| **System Settings Access** | Reading global config parameters (`ir.config_parameter`). | Read-only access: `self.env['ir.config_parameter'].sudo().get_param(...)`. |
| **Automated System Tasks** | Background jobs or crons operating without a logged-in user. | Verify the cron is secured and does not process unvalidated external inputs. |
| **Audit Log Creation** | Creating tracking entries that standard users lack create rights for. | Create the log entry in sudo mode after verifying the action itself. |

---

## 5. Security Evolution: Odoo 18 vs Odoo 20

| Security Feature | Odoo 18 (Current LTS) | Odoo 20 (Future Architecture) |
| :--- | :--- | :--- |
| **Model Access File** | `security/ir.model.access.csv` | Unified `security/ir.access.csv` (`ir.access`) |
| **Multi-Company Auto-Check** | `_check_company_auto = True` | Built-in by default |
| **Multi-Company Rules** | `('company_id', 'in', allowed_company_ids)` | Same |
| **Programmatic Check** | `check_access_rights('write')`<br/>`check_access_rule('write')` | Unified: `recordset.check_access('write')` |
| **Superuser Check** | `self.env.is_superuser()` / `self.env.is_admin()` | Same |

---

## 6. Security Audit Checklist for PRs / Code Reviews

- [x] **No Blind `sudo()` on User IDs**: Every `sudo().browse(id)` derived from user input is preceded by `check_access_rule()`.
- [x] **No Business `sudo()` in Loops**: Avoid `for r in records: r.sudo().write(...)` which ruins performance and safety.
- [x] **Sensitive Fields Protected in Python**: Fields containing financial, margin, or personal data have `groups="..."` in Python, not just XML.
- [x] **Multi-Company Guard**: Multi-company models define `_check_company_auto = True` and use `allowed_company_ids` in `ir.rule`.

