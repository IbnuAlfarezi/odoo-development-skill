# Asynchronous Jobs (OCA queue_job) & Webhook Security Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ASYNC QUEUE_JOB & SECURE WEBHOOK PATTERNS                                   ║
║  Background processing with OCA queue_job, retry logic, HMAC-SHA256 webhooks ║
║  and Odoo 18 JSON-2 API Bearer authentication                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

In enterprise production deployments:
1. **Long-running tasks** (PDF generation, bulk imports, external ERP sync) must be offloaded to **OCA `queue_job`** background workers to prevent web worker timeouts and UI freezing.
2. **Public incoming webhooks** must verify cryptographic signatures (**HMAC-SHA256**) and enforce **Idempotency Keys** to prevent spoofing and duplicate execution.
3. **Machine-to-machine integrations in Odoo 18+** should utilize the official **JSON-2 API** specification with Bearer API Keys.

---

## 1. Asynchronous Background Jobs (OCA `queue_job`)

The OCA `queue_job` addon provides a robust, multi-threaded background queue system with dedicated worker channels, priority handling, and automatic retries.

### Step 1: Decorate the Method with `@job`

```python
from odoo import api, fields, models
from odoo.addons.queue_job.job import job


class StockInventorySync(models.Model):
    _name = "stock.inventory.sync"
    _description = "Inventory External Sync"

    state = fields.Selection([
        ("draft", "Draft"),
        ("queued", "Queued"),
        ("done", "Synchronized"),
        ("failed", "Failed"),
    ], default="draft")

    @job(default_channel="root.data_sync", retry_pattern={1: 60, 2: 180, 3: 600})
    def run_heavy_sync(self, warehouse_id: int):
        """Heavy background task executed by a queue_job worker.
        Auto-retries after 1 min, 3 mins, and 10 mins if an unexpected network error occurs.
        """
        self.ensure_one()
        warehouse = self.env["stock.warehouse"].browse(warehouse_id)
        
        # Execute heavy sync logic without blocking the web client
        self._sync_external_inventory_api(warehouse)
        self.write({"state": "done"})
```

### Step 2: Enqueue with `.with_delay()`

Trigger the background execution from any UI action, controller, or workflow:

```python
    def action_start_background_sync(self):
        self.ensure_one()
        self.write({"state": "queued"})

        # Enqueue immediately to the background job worker
        # priority: 1 (highest) to 100 (lowest)
        self.with_delay(
            priority=15,
            description=f"Sync Inventory for {self.display_name}",
        ).run_heavy_sync(warehouse_id=1)

        return {
            "type": "ir.actions.client",
            "tag": "display_notification",
            "params": {
                "title": "Job Enqueued",
                "message": "Inventory synchronization started in the background.",
                "type": "info",
                "sticky": False,
            },
        }
```

---

## 2. Secure Webhook Receiver (HMAC-SHA256 & Idempotency)

When exposing public webhooks (e.g. from payment gateways, marketplace webhooks, or IoT devices), authenticate the payload using HMAC-SHA256 signature verification.

```python
import hashlib
import hmac
import json
import logging
from odoo import http
from odoo.http import request

_logger = logging.getLogger(__name__)


class ExternalWebhookController(http.Controller):

    def _verify_hmac_signature(self, raw_body: bytes, signature_header: str, secret_key: str) -> bool:
        """Verify HMAC-SHA256 signature against the raw request body."""
        if not signature_header or not secret_key:
            return False
        computed_sig = hmac.new(
            secret_key.encode("utf-8"),
            raw_body,
            hashlib.sha256
        ).hexdigest()
        return hmac.compare_digest(computed_sig, signature_header)

    @http.route("/api/v1/webhook/order_event", type="http", auth="public", methods=["POST"], csrf=False)
    def receive_order_webhook(self, **kwargs):
        raw_body = request.httprequest.data
        signature = request.httprequest.headers.get("X-Signature-SHA256")
        idempotency_key = request.httprequest.headers.get("X-Idempotency-Key")

        # 1. Fetch secret key from System Parameters
        secret = request.env["ir.config_parameter"].sudo().get_param("external_api.webhook_secret")

        # 2. Cryptographic signature check
        if not self._verify_hmac_signature(raw_body, signature, secret):
            _logger.warning("Unauthorized webhook attempt from IP %s", request.httprequest.remote_addr)
            return request.make_json_response({"error": "Invalid signature"}, status=401)

        # 3. Prevent duplicate execution using Idempotency Key
        if idempotency_key:
            existing_log = request.env["webhook.event.log"].sudo().search([
                ("idempotency_key", "=", idempotency_key)
            ], limit=1)
            if existing_log:
                return request.make_json_response({"status": "already_processed"}, status=200)

        # 4. Parse JSON payload safely
        try:
            payload = json.loads(raw_body.decode("utf-8"))
        except (ValueError, UnicodeDecodeError):
            return request.make_json_response({"error": "Malformed JSON"}, status=400)

        # 5. Offload processing to background queue_job
        log = request.env["webhook.event.log"].sudo().create({
            "idempotency_key": idempotency_key,
            "event_type": payload.get("event"),
            "payload_data": raw_body.decode("utf-8"),
        })
        log.with_delay(priority=20).process_webhook_event()

        return request.make_json_response({"status": "received", "log_id": log.id}, status=200)
```

---

## 3. Odoo 18 JSON-2 API with Bearer Token

Starting with Odoo 18, external integrations can communicate directly with the official `/json/2/` API endpoint using Bearer API Keys generated from user profiles.

### External Client Request Pattern (Python Client)

```python
import requests

BASE_URL = "https://mycompany.odoo.com/json/2"
API_KEY = "6578616d706c65206a736f6e20617069206b6579"

HEADERS = {
    "Authorization": f"bearer {API_KEY}",
    "X-Odoo-Database": "production_db",
    "Content-Type": "application/json",
}

# 1. Search Partner IDs
search_res = requests.post(
    f"{BASE_URL}/res.partner/search",
    headers=HEADERS,
    json={
        "domain": [["customer_rank", ">", 0], ["is_company", "=", True]],
        "limit": 10,
    },
)
partner_ids = search_res.json()

# 2. Read Partner Data
read_res = requests.post(
    f"{BASE_URL}/res.partner/read",
    headers=HEADERS,
    json={"ids": partner_ids, "fields": ["name", "email", "phone"]},
)
partners = read_res.json()
print("Retrieved partners:", partners)

# 3. Create Record
create_res = requests.post(
    f"{BASE_URL}/sale.order/create",
    headers=HEADERS,
    json={"values": {"partner_id": partner_ids[0]}},
)
new_order_id = create_res.json()
```

---

## 4. Key Rules & Decision Matrix

| Scenario | Recommended Approach | Key Component |
| :--- | :--- | :--- |
| **Heavy background process (> 5 seconds)** | OCA `queue_job` | `@job` + `with_delay()` |
| **Periodic scheduled task (e.g. daily invoice)** | Odoo Core Cron | `ir.cron` with `model.method()` |
| **Public incoming webhook** | Signed Controller | `@http.route(auth='public')` + HMAC-SHA256 validation |
| **Machine-to-machine API integration** | Odoo 18 JSON-2 API | `POST /json/2/<model>/<method>` with Bearer API Key |

