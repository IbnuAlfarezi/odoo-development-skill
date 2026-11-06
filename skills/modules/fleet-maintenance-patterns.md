# Fleet Management & Equipment Maintenance Patterns

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  FLEET MANAGEMENT & EQUIPMENT MAINTENANCE PATTERNS                           ║
║  Vehicle tracking, Mileage logs, Servicing costs (fleet.vehicle),            ║
║  Preventive vs Corrective equipment maintenance (maintenance.request)        ║
║  Covers Odoo 16.0, 17.0, 18.0, 19.0, and 20.0                                ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

Odoo manages company assets and physical operations through two operational modules:
1. **`fleet`**: Manages company vehicles, fuel consumption, mileage/odometer readings, recurring leasing contracts, and repair costs.
2. **`maintenance`**: Manages internal machinery, tools, and work centers. It coordinates **preventive** (scheduled checkups) and **corrective** (breakdown repairs) maintenance requests, integrating directly with Manufacturing (MRP) work centers.

---

## 1. Fleet Vehicle Lifecycle (`fleet.vehicle`)

A vehicle record tracks physical specifications, driver assignments, and running operating costs:

```python
from odoo import api, fields, models


class FleetVehicle(models.Model):
    _inherit = 'fleet.vehicle'

    gps_tracker_id = fields.Char(string="GPS Unit ID", index=True)
    fuel_efficiency_target = fields.Float(string="Target Fuel Consumption (L/100km)", default=8.0)
    service_urgency = fields.Selection([
        ('normal', "Normal"),
        ('due_soon', "Service Due Soon"),
        ('overdue', "Service Overdue"),
    ], string="Service Urgency", compute="_compute_service_urgency", store=True)

    @api.depends('odometer', 'log_services.odometer')
    def _compute_service_urgency(self):
        """Evaluate if vehicle requires scheduled maintenance based on mileage."""
        for vehicle in self:
            last_service_odometer = max(
                vehicle.log_services.mapped('odometer') or [0.0]
            )
            mileage_since_service = vehicle.odometer - last_service_odometer
            if mileage_since_service >= 10000:
                vehicle.service_urgency = 'overdue'
            elif mileage_since_service >= 8500:
                vehicle.service_urgency = 'due_soon'
            else:
                vehicle.service_urgency = 'normal'
```

---

## 2. Vehicle Servicing & Repair Logging (`fleet.vehicle.log.services`)

Service logs record maintenance performed by internal mechanics or external vendors, tracking costs and parts replaced:

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError


class FleetVehicleLogServices(models.Model):
    _inherit = 'fleet.vehicle.log.services'

    warranty_claim = fields.Boolean(string="Claimed Under Warranty", default=False)
    vendor_invoice_number = fields.Char(string="Vendor Repair Invoice #")

    def action_done(self):
        """Complete service log and validate costs."""
        for service in self:
            if service.cost <= 0.0 and not service.warranty_claim:
                raise UserError(_(
                    "Service for vehicle %s must specify a positive cost unless flagged as a warranty claim."
                ) % service.vehicle_id.name)

            # Auto-update vehicle odometer if service reading is higher
            if service.odometer > service.vehicle_id.odometer:
                service.vehicle_id.odometer = service.odometer

        return super().action_done()
```

---

## 3. Equipment & Machine Maintenance (`maintenance.equipment`)

Tracks machines, computers, and factory tools:

```python
from odoo import api, fields, models


class MaintenanceEquipment(models.Model):
    _inherit = 'maintenance.equipment'

    manufacturer_model_code = fields.Char(string="Manufacturer Model Code")
    operating_hours = fields.Float(string="Total Operating Hours", default=0.0)
    requires_safety_lockout = fields.Boolean(
        string="Lockout/Tagout (LOTO) Required",
        help="Enforces safety shutdown protocols during maintenance.",
    )
```

---

## 4. Maintenance Requests (`maintenance.request`)

Maintenance requests track issue resolution across stages (*New Request* → *In Progress* → *Repaired* → *Scrap*):

### Maintenance Types:
- **`corrective`**: Unscheduled breakdown repair.
- **`preventive`**: Regularly scheduled servicing to avoid failure.

```python
from odoo import _, api, fields, models
from odoo.exceptions import UserError


class MaintenanceRequest(models.Model):
    _inherit = 'maintenance.request'

    safety_lockout_completed = fields.Boolean(string="LOTO Safety Completed", tracking=True)

    @api.onchange('equipment_id')
    def _onchange_equipment_id(self):
        """Auto-populate team and work center from equipment."""
        if self.equipment_id:
            self.maintenance_team_id = self.equipment_id.maintenance_team_id
            if hasattr(self.equipment_id, 'workcenter_id'):
                self.workcenter_id = self.equipment_id.workcenter_id

    def archive_equipment_done(self):
        """Transition request to Repaired stage."""
        for request in self:
            # Enforce safety check if equipment requires Lockout/Tagout
            if request.equipment_id.requires_safety_lockout and not request.safety_lockout_completed:
                raise UserError(_(
                    "Safety protocol violation: Equipment '%s' requires Lockout/Tagout (LOTO) "
                    "confirmation before closing this maintenance request."
                ) % request.equipment_id.name)

            # If maintenance was blocking a manufacturing work center, unblock it
            if request.workcenter_id and request.block_workcenter:
                request.workcenter_id.working_state = 'normal'

        return super().archive_equipment_done()
```

---

## 5. Manufacturing (MRP) Integration: Halting Work Centers on Breakdown

When a piece of machinery breaks down during production, the maintenance request can automatically halt scheduled work orders on that work center:

```python
from odoo import api, models


class MaintenanceRequest(models.Model):
    _inherit = 'maintenance.request'

    def action_block_associated_workcenter(self):
        """Mark work center as blocked to stop manufacturing scheduling."""
        self.ensure_one()
        if self.workcenter_id:
            self.block_workcenter = True
            self.workcenter_id.write({
                'working_state': 'blocked',
            })
            self.workcenter_id.message_post(body=_(
                "Work Center BLOCKED due to emergency maintenance request: %s"
            ) % self.name)
```

---

## 6. Best Practices Checklist for Fleet & Maintenance

1. **Odometer Unit Consistency**: Always verify `vehicle.odometer_unit` (`kilometers` vs `miles`) before calculating mileage intervals or fuel economy ratios.
2. **Scheduled Preventative Recurrence**: Set `period` (in days) on `maintenance.equipment` so Odoo automatically schedules the next recurring preventive maintenance request upon completion.
3. **Audit Trail**: Use Odoo's Chatter (`mail.thread`) on both `fleet.vehicle` and `maintenance.request` to track stage transitions, costs, and technician assignments.

