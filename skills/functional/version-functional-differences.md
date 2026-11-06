# Odoo Functional Evolution & Version Differences Matrix (v14 – v20)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  ODOO FUNCTIONAL EVOLUTION & VERSION DIFFERENCES (v14 - v20)                 ║
║  Comprehensive Matrix of Business Process Changes, UI Overhauls & New Models ║
║  Essential Guide for Migration, Functional Consulting & Version Upgrades     ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

## Overview

While the fundamental double-entry principles of Odoo ERP remain consistent across versions, **business workflows, document models, configurations, and user interfaces undergo major structural evolution** from version 14 to version 20.

This document serves as the master reference mapping exactly **how functional processes changed** between major Odoo releases.

---

## 1. Executive Summary: Major Functional Turning Points

```
v14 / v15 (Classic Era)
  ├── Separate sale.subscription model & legacy templates
  ├── Single analytic account per line (analytic_account_id / analytic_tag)
  ├── Legacy JS bank reconciliation widget
  └── Standard backend views for Shop Floor operators
       │
       ▼ (v16 Major Overhaul)
v16 (Unified Architecture)
  ├── Subscriptions merged directly into sale.order (recurring_rule_id)
  ├── Analytic Plans introduced (analytic_distribution JSON matrix)
  ├── Bank Reconciliation completely rewritten as embedded list/form
  └── Shop Floor tablet app introduced for Manufacturing work orders
       │
       ▼ (v17 - v18 Modernization)
v17 / v18 (Speed & Automation)
  ├── 1-Page Express eCommerce checkout & mobile express payment buttons
  ├── Native Global Peppol / UBL / Factur-X electronic invoicing
  ├── Real-time barcode app overhaul (touch-optimized)
  └── Hoot modern testing framework replacing legacy QUnit
       │
       ▼ (v19 - v20 Modern Enterprise)
v19 / v20 (Modern Architecture)
  ├── ir.access model architecture replacing legacy ir.model.access
  ├── OWL 3 frontend reactive rendering
  ├── group_ids replacing groups_id
  └── Native automated AI reconciliation & document OCR parsing
```

---

## 2. Domain-by-Domain Version Comparison

### A. Sales & Subscriptions

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **Subscription Architecture** | Dedicated `sale.subscription` model with separate contract numbers. | **Merged into `sale.order`**: Any SO becomes a subscription if recurring product added. | Native recurring rules with pro-rated mid-month alignment. | Self-service upgrade/downgrade via customer portal with payment tokenization. |
| **Pricelist Computation** | Legacy calculation with distinct margin/fixed/percentage formulas. | Unified pricelist engine with rule priority. | Discount display directly on ecommerce product grids. | Dynamic real-time customer currency conversion. |
| **Sales Order Confirmation** | Confirmation button creates pickings immediately. | Option to validate online payment before confirmation. | Direct express checkout without quotation intermediate. | Biometric / One-touch approvals via portal. |

---

### B. Accounting, Invoicing & Bank Reconciliation

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **Bank Reconciliation** | Legacy JavaScript widget (`account.reconciliation.widget`). | **Rewritten**: Bank reconciliation integrated directly as form view with auto-matching tabs. | Auto-reconciliation models with rule-based tolerances. | AI-driven batch matching and automated statement import integrations. |
| **Analytic Accounting** | Single analytic account per line (`analytic_account_id`) or rigid tags. | **Analytic Plans (`analytic.plan`)**: Multi-dimensional distribution via `analytic_distribution` (JSON). | Mandatory analytic distribution validation per account. | Dynamic cost reallocation models across business units. |
| **E-Invoicing / EDI** | Basic XML export, manual local compliance modules. | European Peppol / UBL integration built into core. | Native global Peppol directory lookup and automated dispatch. | Mandatory international B2B electronic compliance by default. |
| **Financial Reports** | Static PDF generation or basic dynamic drill-down. | Fast dynamic spreadsheet-style drill-down reports. | Customizable analytical financial filters in UI. | Instant real-time multi-currency consolidation across subsidiaries. |

---

### C. Purchasing & Inbound Logistics (P2P)

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **3-Way Matching** | Manual comparison or basic warning banner on vendor bill. | Native 3-Way Matching exception status flag on bill line. | Automated bill parsing from PDF/XML with automated PO matching. | Automated vendor bill auto-posting when 3-way match is 100% clean. |
| **Purchase Agreements** | Tender vs Blanket orders in separate views. | Unified purchase agreement types with auto-selection. | Direct vendor portal bidding and quotation submission. | Multi-tier approval rules based on purchase amount thresholds. |

---

### D. Inventory, Warehousing & Valuation

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **Physical Inventory (Opname)** | Periodic inventory wizard generating stock adjustments. | Continuous inventory count directly on `stock.quant` list view. | Real-time mobile barcode scan for cycle counts. | Automated discrepancy alert triggers and inventory auditor sign-off. |
| **Landed Costs** | Standalone record manually linked to receipt pickings. | Split methods enhanced (weight, volume, value, equal). | Landed costs directly linkable to supplier bills. | Real-time automated landed cost distribution on drop-ship and multi-warehouse transfers. |
| **Packaging & Barcode** | Basic package hierarchy (`stock.quant.package`). | Automated batch picking by delivery carrier / zone. | Wave picking and cluster picking native in Barcode app. | Touch-first warehouse kiosk interface with visual bin location mapping. |

---

### E. Manufacturing (MRP) & Shop Floor

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **Operator Interface** | Standard backend form views or barcode app. | **Shop Floor App (`mrp_workorder`)**: Touchscreen tablet UI for work center operators. | Interactive multi-step assembly guides with video/slide integration. | Real-time work center maintenance blocking directly from tablet screen. |
| **Subcontracting** | Basic subcontracting BOM type. | Subcontractor portal for raw material receipt tracking. | Dropshipping components directly to subcontractor with lot tracking. | Automated subcontractor production verification. |
| **BOM Management** | Static multi-level BOMs. | BOM revisions and Engineering Change Orders (ECO). | By-product allocation percentage customization. | Dynamic component substitution based on real-time raw material availability. |

---

### F. Human Resources & Payroll (H2R)

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **Time Off Accrual** | Basic allocation granting schedules. | **Accrual Plans (`hr.leave.accrual.plan`)**: Milestone tiers, carryover limits, and pro-rating. | Public holiday auto-exclusion and optional religious holidays. | Negative balance threshold controls and automatic overtime-to-leave conversion. |
| **Security & Access** | Employee groups assigned via `groups_id`. | Employee profile linked to user and partner records. | Enhanced private data encryption for employee contracts. | **`group_ids` architecture**: Clean access rule assignments. |
| **Appraisals & Feedback** | Annual formal evaluation form. | Continuous 360-degree feedback loops with peer reviews. | Skill matrix and employee skill level progression tracking. | Automated performance milestones linked to compensation reviews. |

---

### G. Retail, POS & eCommerce

| Functional Feature | Odoo 14 – 15 | Odoo 16 | Odoo 17 – 18 | Odoo 19 – 20 |
|---|---|---|---|---|
| **POS Architecture** | Legacy backbone/custom frontend rendering. | Full OWL component architecture in POS UI. | Instant kitchen order routing & customer-facing display support. | Self-ordering kiosk mode and QR-code table ordering. |
| **eCommerce Checkout** | 3-step checkout (Address ➔ Shipping ➔ Payment). | Streamlined 2-step checkout with guest support. | 1-page express checkout with integrated shipping rates. | One-click buy buttons (Apple Pay, Google Pay) and real-time stock bin availability. |

---

## 3. Migration & Version Coexistence Guidelines

When consulting on or implementing functional flows across different Odoo versions:

1. **Never assume Subscriptions are separate in v16+**: Always configure them via `sale.order` with `plan_id`.
2. **Never look for `analytic_account_id` on invoice lines in v16+**: Always inspect the `analytic_distribution` dictionary.
3. **Never attempt to customize the old JS reconciliation widget in v16+**: Build custom reconciliation logic using `account.reconcile.model` rules.
4. **Always verify whether client runs Community or Enterprise**:
   - Dynamic Financial Reports, Bank Sync, Quality Control Points, Shop Floor App, and Helpdesk SLAs are Enterprise core features.
   - Core sales, procurement, inventory double-entry, basic accounting, and manufacturing are fully functional in Community.

