# HR Advanced: Time Off (`hr_holidays`) & Recruitment (`hr_recruitment`) Patterns

## Overview & Architecture

Beyond basic employee directory records (`hr.employee`), human resources in enterprise Odoo relies heavily on two interconnected operational engines:
1. **Time Off & Leave Management (`hr_holidays`)**: Regulates employee time off requests, multi-tier approvals, public calendar collision detection, and automated accrual balance calculations.
2. **Talent Acquisition & Recruitment Pipeline (`hr_recruitment`)**: Tracks applicants across pipeline stages from incoming resumes to candidate evaluation, salary negotiation, and automated transition into active `hr.employee` records.

```
       [Talent Acquisition: hr_recruitment]
Applicant Submits Resume ──> hr.applicant (Stage Pipeline)
                                  │
                                  ▼ action_create_employee()
                            hr.employee (Active Staff)
                                  │
       [Time Off Engine: hr_holidays]
                                  │
        ┌─────────────────────────┴─────────────────────────┐
        ▼                                                   ▼
hr.leave.allocation                                     hr.leave
(Regular or Accrual Plan)                     (Time Off Request & Approval)
        │                                                   │
        └────────────────── Balance Check ──────────────────┘
```

---

## 1. Time Off Architecture (`hr_holidays`)

### Core Models

| Model | Purpose | Key Fields |
|---|---|---|
| `hr.leave.type` | Defines rules, validity, and approval tiers for leave types (Annual, Sick, Maternity). | `leave_validation_type`, `requires_allocation`, `request_unit`, `max_leaves` |
| `hr.leave.allocation` | Grants time off balance to employees either individually or in batches. | `holiday_status_id`, `employee_id`, `holiday_type`, `allocation_type`, `number_of_days` |
| `hr.leave` | Individual employee request to take time off. | `holiday_status_id`, `employee_id`, `date_from`, `date_to`, `number_of_days`, `state` |
| `hr.leave.accrual.plan` | Automated recurring schedule granting balance over time (e.g., 1.5 days/month). | `level_ids`, `accrued_gain_time` |

### Leave Type Configuration (`hr.leave.type`)

```python
from odoo import api, fields, models, _
from odoo.exceptions import ValidationError

class HrLeaveType(models.Model):
    _inherit = "hr.leave.type"

    # Validation requirements:
    # 'no_validation' -> Auto approved
    # 'hr'            -> Time Off Officer only
    # 'manager'       -> Employee's direct manager only
    # 'both'          -> Manager approval first, then HR Officer validation
    #
    # Allocation requirement:
    # 'yes'           -> Request requires pre-allocated balance
    # 'no'            -> Request can be submitted without prior allocation
```

### Allocation Lifecycle & Group Grants (`hr.leave.allocation`)

Allocations can be granted to individual employees or grouped across departments, tags, or entire companies:

```python
from odoo import api, fields, models, _

class HrLeaveAllocation(models.Model):
    _inherit = "hr.leave.allocation"

    def action_grant_company_annual_leave(self, company_id, leave_type_id, days):
        """Programmatically create group allocation for all active employees in a company."""
        allocation = self.create({
            "name": _("Annual Allocation %s", fields.Date.today().year),
            "holiday_status_id": leave_type_id,
            "holiday_type": "company",
            "mode_company_id": company_id,
            "allocation_type": "regular",
            "number_of_days": days,
        })
        # Moves draft -> confirm -> validate
        allocation.action_confirm()
        allocation.action_validate()
        return allocation
```

### Leave Request State Machine & Approvals (`hr.leave`)

The standard leave lifecycle progresses through:
- `draft`: Request created by employee or HR.
- `confirm`: Submitted for approval.
- `validate1`: First approval granted (if `leave_validation_type == 'both'`).
- `validate`: Fully approved; calendar meeting created and balance deducted.
- `refuse`: Rejected by manager or HR officer.

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError, ValidationError

class HrLeave(models.Model):
    _inherit = "hr.leave"

    def action_approve(self):
        """Override approval to enforce mandatory handover documentation or audit logs."""
        for leave in self:
            if leave.number_of_days > 5 and not leave.name:
                raise UserError(_("Time off requests exceeding 5 days require a reason / handover note."))
        return super().action_approve()

    @api.constrains("date_from", "date_to", "employee_id")
    def _check_blackout_periods(self):
        """Prevent scheduling leave during critical operational blackout dates."""
        for leave in self:
            if not leave.date_from or not leave.date_to:
                continue
            # Example: Year-end financial close blackout
            year = leave.date_from.year
            blackout_start = fields.Datetime.to_datetime(f"{year}-12-25 00:00:00")
            blackout_end = fields.Datetime.to_datetime(f"{year}-12-31 23:59:59")
            
            if (leave.date_from <= blackout_end) and (leave.date_to >= blackout_start):
                if leave.employee_id.department_id.name == "Finance":
                    raise ValidationError(_("Leaves during Year-End Finance Close (Dec 25-31) are restricted."))
```

---

## 2. Recruitment Pipeline Architecture (`hr_recruitment`)

### Core Models

| Model | Purpose | Key Fields |
|---|---|---|
| `hr.job` | Target job position open for hiring. | `name`, `no_of_recruitment`, `no_of_hired_employee`, `stage_ids` |
| `hr.recruitment.stage` | Pipeline stages (Initial, Interview 1, Contract Proposal, Hired). | `name`, `sequence`, `fold`, `hired_stage` |
| `hr.applicant` | Candidate applications, resume attachments, scores, and stages. | `partner_name`, `email_from`, `stage_id`, `job_id`, `salary_expected`, `emp_id` |

### Applicant Progression & Automation

```python
from odoo import api, fields, models, _
from odoo.exceptions import UserError

class HrApplicant(models.Model):
    _inherit = "hr.applicant"

    internal_notes = fields.Text(string="Technical Interview Feedback")
    candidate_rating = fields.Selection([
        ("1", "Basic"),
        ("2", "Good"),
        ("3", "Strong"),
        ("4", "Exceptional"),
    ], string="Evaluation Score")

    def action_promote_to_technical_interview(self):
        """Move applicant to technical interview stage and notify interview panel."""
        tech_stage = self.env["hr.recruitment.stage"].search(
            [("name", "ilike", "Technical Interview")], limit=1
        )
        if not tech_stage:
            raise UserError(_("Technical Interview stage not configured."))
            
        self.write({"stage_id": tech_stage.id})
        for applicant in self:
            applicant.message_post(
                body=_("Candidate moved to Technical Interview stage. Evaluation required."),
                subtype_xmlid="mail.mt_comment",
            )
```

### Hiring & Employee Conversion Flow (`create_employee_from_applicant`)

When an applicant is hired, Odoo converts the record into an active `hr.employee`:

```python
class HrApplicant(models.Model):
    _inherit = "hr.applicant"

    def create_employee_from_applicant(self):
        """Customized employee creation preserving applicant metadata and attachments."""
        action = super().create_employee_from_applicant()
        
        # Super returns an act_window action dictionary opening the new employee form
        employee_id = action.get("res_id")
        if employee_id:
            employee = self.env["hr.employee"].browse(employee_id)
            for applicant in self:
                # Synchronize expected / agreed salary to private employee contract fields
                employee.write({
                    "notes": applicant.internal_notes,
                })
                
                # Copy attached resume / certifications from applicant to employee
                attachments = self.env["ir.attachment"].search([
                    ("res_model", "=", "hr.applicant"),
                    ("res_id", "=", applicant.id),
                ])
                for att in attachments:
                    att.copy({
                        "res_model": "hr.employee",
                        "res_id": employee.id,
                    })
        return action
```

---

## 3. Cross-Module Integration: Automatic Onboarding & Leave Initialization

Upon hiring a candidate through `hr_recruitment`, standard enterprise workflows trigger allocation initialization in `hr_holidays`:

```python
class HrEmployee(models.Model):
    _inherit = "hr.employee"

    @api.model_create_multi
    def create(self, vals_list):
        employees = super().create(vals_list)
        # Automatically assign pro-rated onboarding annual leaves
        annual_leave_type = self.env["hr.leave.type"].search([
            ("name", "ilike", "Annual Leave"),
            ("requires_allocation", "=", "yes"),
        ], limit=1)

        if annual_leave_type:
            current_month = fields.Date.today().month
            remaining_months = 13 - current_month
            pro_rated_days = round(remaining_months * 1.5, 1)  # 1.5 days/month standard

            for employee in employees:
                self.env["hr.leave.allocation"].create({
                    "name": _("Initial Pro-rated Leave (%s)", fields.Date.today().year),
                    "employee_id": employee.id,
                    "holiday_status_id": annual_leave_type.id,
                    "allocation_type": "regular",
                    "number_of_days": pro_rated_days,
                    "state": "validate",  # Pre-approved onboarding grant
                })
        return employees
```

---

## 4. Common Pitfalls & Prevention

| Pitfall | Risk | Recommended Solution |
|---|---|---|
| **Datetime vs Date timezone mismatch** | Leave days miscalculated or span shifted across midnight UTC. | Always compute leave days utilizing employee calendar: `resource_calendar_id._work_intervals_batch(from_dt, to_dt)`. |
| **Bypassing State Approvals via `write({'state': 'validate'})`** | Leaves marked approved without generating calendar meetings, mail threads, or updating accrual counters. | Always invoke standard lifecycle methods: `leave.action_approve()` or `leave.action_validate()`. |
| **Orphaned Attachments upon Hiring** | Resumes and reference checks attached to `hr.applicant` remain inaccessible from `hr.employee`. | Copy `ir.attachment` records referencing `hr.applicant` over to the newly generated `hr.employee` record. |
| **Accrual Overdraw Without Caps** | Employees exhaust excessive time off beyond allowable negative balances. | Configure `max_leaves` or enforce validation constraints on negative balance thresholds in `hr.leave.type`. |

