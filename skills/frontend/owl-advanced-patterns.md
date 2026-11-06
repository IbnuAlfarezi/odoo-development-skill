# Advanced OWL Component & Frontend Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ADVANCED OWL FRONTEND PATTERNS                                              ║
║  Component patching, realtime WebSocket bus notifications, custom services,  ║
║  and custom view controller extensions (js_class)                            ║
║  Covers Odoo 16.0, 17.0, 18.0, and 20.0                                      ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

While basic OWL development covers standalone components and field widgets, enterprise-grade Odoo modules frequently require modifying standard core behavior without replacing core templates. 

This guide provides tested, production-ready patterns for:
1. **Monkey Patching (`patch`)**: Modifying core controllers, renderers, and components.
2. **Realtime WebSocket Bus (`bus_service`)**: Receiving live server push notifications without page refresh.
3. **Custom Services (`services` Registry)**: Creating reusable global services accessible across components.
4. **View Extension via `js_class`**: Extending standard Form/List controllers and binding them in XML.

---

## 1. Component & Class Patching (`patch()`)

The `patch` utility from `@web/core/utils/patch` allows in-place extension of JavaScript classes and OWL components. Patched methods can call original implementations using the native `super` keyword.

### Version Differences in `patch()` Signature

| Odoo Version | Signature | Example |
| :--- | :--- | :--- |
| **v16.0** | `patch(target, patchName, patchObject)` | `patch(FormController.prototype, "my_module.patch", { ... })` |
| **v17.0 & v18.0** | `patch(target, patchObject)` *(String name removed)* | `patch(FormController.prototype, { ... })` |
| **v20.0** | `patch(target, patchObject)` *(Discouraged for new code in favor of plugins, but fully backward compatible)* | `patch(FormController.prototype, { ... })` |

### Pattern: Patching `FormController` (Odoo 17 & 18)

Use this pattern to intercept form saving or inject custom hooks into standard forms:

```javascript
/** @odoo-module **/

import { FormController } from "@web/views/form/form_controller";
import { patch } from "@web/core/utils/patch";
import { useService } from "@web/core/utils/hooks";

patch(FormController.prototype, {
    setup() {
        super.setup();
        this.notification = useService("notification");
    },

    async beforeSave() {
        // Intercept save operations across all form views
        console.log("Saving record:", this.model.root.resId);
        
        // Always call super to preserve standard validation & save pipeline
        return super.beforeSave(...arguments);
    },
});
```

### Critical Rules for Patching:
1. **Always call `super.methodName(...arguments)`** to maintain core Odoo behavior unless explicitly intending an override.
2. **Never patch constructors directly.** Patch `setup()` for components or initialization methods.
3. **Apply patches at the top level** of your ES module file so they execute as soon as assets load.

---

## 2. Realtime WebSocket Bus Notifications (`bus_service` & `bus.bus`)

Odoo includes a WebSocket-based push notification system. Backend Python code can push live events to connected web clients instantly without polling.

### Step 1: Server-Side Dispatch (Python Backend)

```python
from odoo import api, models


class ProjectTask(models.Model):
    _inherit = "project.task"

    def action_complete_task(self):
        self.ensure_one()
        self.state = "done"

        # Push notification to the task assignee
        if self.user_id:
            target_partner = self.user_id.partner_id
            payload = {
                "task_id": self.id,
                "task_name": self.name,
                "message": f"Task '{self.name}' has been marked as completed!",
                "type": "success",
            }
            # _sendone(channel_or_partner, notification_type, message)
            self.env["bus.bus"]._sendone(
                target_partner,
                "project_task_update",
                payload,
            )
```

### Step 2: Client-Side Listener Service (JavaScript Frontend)

Create a dedicated frontend service that subscribes to the channel on startup and triggers an action or toast:

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";

export const taskNotificationService = {
    dependencies: ["bus_service", "notification", "action"],
    start(env, { bus_service, notification, action }) {
        // Subscribe to the notification type emitted by Python _sendone
        bus_service.subscribe("project_task_update", (payload) => {
            notification.add(payload.message, {
                title: "Task Completed",
                type: payload.type || "info",
                sticky: false,
                buttons: [
                    {
                        name: "Open Task",
                        primary: true,
                        onClick: () => {
                            action.doAction({
                                type: "ir.actions.act_window",
                                res_model: "project.task",
                                res_id: payload.task_id,
                                views: [[false, "form"]],
                                target: "current",
                            });
                        },
                    },
                ],
            });
        });

        // Ensure the bus connection is active
        bus_service.start();
    },
};

registry.category("services").add("task_notification_service", taskNotificationService);
```

### Step 3: Listening Inside a Specific Component with `useBus`

If a specific component only needs to react while mounted on screen, use the `useBus` hook (automatically unsubscribes on unmount):

```javascript
import { Component } from "@odoo/owl";
import { useBus } from "@web/core/utils/hooks";

export class TaskKanbanDashboard extends Component {
    static template = "my_module.TaskKanbanDashboard";

    setup() {
        // Automatically binds to env.bus and cleans up on unmount
        useBus(this.env.bus, "project_task_update", (ev) => {
            console.log("Live task update received:", ev.detail);
            this.refreshDashboard();
        });
    }

    refreshDashboard() {
        // Reload dashboard state
    }
}
```

---

## 3. Custom Global Services (`services` Registry)

Services in Odoo are singletons that provide shared state, network connections, or business utilities across any OWL component via `useService()`.

### Pattern: Defining a Custom Shared Service

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { reactive } from "@odoo/owl";

export const activeShiftService = {
    dependencies: ["orm", "notification"],
    async start(env, { orm, notification }) {
        // Shared reactive state across all components
        const state = reactive({
            currentShift: null,
            isOnline: true,
        });

        async function fetchCurrentShift(userId) {
            try {
                const shift = await orm.searchRead(
                    "mining.shift",
                    [["user_id", "=", userId], ["state", "=", "active"]],
                    ["name", "start_time"],
                    { limit: 1 }
                );
                state.currentShift = shift[0] || null;
            } catch (err) {
                notification.add("Could not fetch active shift: " + err.message, {
                    type: "danger",
                });
            }
        }

        // Return public API of the service
        return {
            state,
            fetchCurrentShift,
        };
    },
};

registry.category("services").add("active_shift", activeShiftService);
```

### Pattern: Consuming Service Inside an OWL Component

```javascript
/** @odoo-module **/

import { Component, onWillStart } from "@odoo/owl";
import { useService } from "@web/core/utils/hooks";

export class ShiftStatusBar extends Component {
    static template = "my_module.ShiftStatusBar";

    setup() {
        this.user = useService("user");
        this.shiftService = useService("active_shift");

        onWillStart(async () => {
            await this.shiftService.fetchCurrentShift(this.user.userId);
        });
    }

    get shiftName() {
        return this.shiftService.state.currentShift?.name || "No Active Shift";
    }
}
```

---

## 4. Custom View Controller Extension via `js_class`

When you need custom buttons, headers, or unique interaction modes on a specific Form or List view, use the `js_class` pattern.

### Step 1: Create Custom Controller & View Object (JavaScript)

```javascript
/** @odoo-module **/

import { registry } from "@web/core/registry";
import { formView } from "@web/views/form/form_view";
import { FormController } from "@web/views/form/form_controller";
import { useService } from "@web/core/utils/hooks";

export class MiningFleetFormController extends FormController {
    setup() {
        super.setup();
        this.action = useService("action");
        this.notification = useService("notification");
    }

    async onPrintInspectionReport() {
        const resId = this.model.root.resId;
        if (!resId) {
            this.notification.add("Please save the record first.", { type: "warning" });
            return;
        }
        return this.action.doAction("my_module.action_report_inspection", {
            additionalContext: { active_ids: [resId] },
        });
    }
}

// Inherit from standard formView, replacing only the Controller
export const miningFleetFormView = {
    ...formView,
    Controller: MiningFleetFormController,
};

// Register in views category with unique identifier
registry.category("views").add("mining_fleet_form", miningFleetFormView);
```

### Step 2: Bind in Architecture XML (`js_class`)

Connect the registered view to your Odoo XML view using the `js_class` attribute:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="view_mining_equipment_form" model="ir.ui.view">
        <field name="name">mining.equipment.form</field>
        <field name="model">mining.equipment</field>
        <field name="arch" type="xml">
            <!-- Connect custom controller via js_class -->
            <form js_class="mining_fleet_form">
                <header>
                    <button name="action_confirm" string="Confirm" type="object" class="btn-primary"/>
                </header>
                <sheet>
                    <group>
                        <field name="name"/>
                        <field name="serial_number"/>
                    </group>
                </sheet>
            </form>
        </field>
    </record>
</odoo>
```

---

## 5. Summary & Decision Matrix

| Requirement | Recommended Pattern | Key Method / API |
| :--- | :--- | :--- |
| **Modify ALL standard forms/lists globally** | Component Patching | `patch(FormController.prototype, { ... })` |
| **Modify a SINGLE specific form/list** | Custom View Controller | `js_class="my_view"` + `registry.category("views").add()` |
| **Share state/logic across multiple components** | Custom Service | `registry.category("services").add()` + `useService()` |
| **Push live server updates to frontend** | Realtime WebSocket Bus | Python `bus.bus._sendone()` + JS `bus_service.subscribe()` |
| **Clean component-scoped bus listener** | Lifecycle Hook | `useBus(this.env.bus, eventName, callback)` |

