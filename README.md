# Max Biocare Raw Materials Procurement

Power Apps Canvas App (Power Fx) managing the end-to-end procurement request workflow for raw materials at Max Biocare.

This repository tracks the **screen logic and UI only** (`Src/*.pa.yaml`) for change history and review. It is **not** a full `.msapp` package — the generated metadata (`Controls/`, `References/`, `Properties.json`, …) is intentionally excluded (see [`.gitignore`](.gitignore)). The source of truth for editing remains **Power Apps Studio**; this repo is for tracing changes over time.

## Repository layout

```
Src/
├── _EditorState.pa.yaml          # Screen order
├── App.pa.yaml                   # App.OnStart — user resolution + global state
├── HomeScreen.pa.yaml            # Role-based request list + status filters
├── RequestFormScreen.pa.yaml     # Requester submits a new request (incl. raw-material line items)
├── RequestDetailScreen.pa.yaml   # Read-only detail + history of a request
├── ManagerReviewScreen.pa.yaml   # Step 2 — department manager decision
├── ExecutiveApprovalScreen.pa.yaml   # Step 3 — executive decision
├── ProcurementExecutionScreen.pa.yaml # Procurement + invoice parsing
├── InvoiceSubmissionScreen.pa.yaml    # Procurement/Admin invoice submission & extraction
├── RequesterInvoiceScreen.pa.yaml     # Requester uploads/re-uploads their invoice
├── AccountingScreen.pa.yaml      # Accounting handling — always closes the request
├── GoodsReceiptScreen.pa.yaml    # Goods receipt & acceptance (per-material batch/expiry)
└── SupplierFollowUpScreen.pa.yaml    # Supplier follow-up (2 rounds, per-material batch/expiry)
```

See [`CLAUDE.md`](CLAUDE.md) for the detailed architecture and conventions.

## Backend

- **SharePoint Online** (`maxbiocare.sharepoint.com/sites/Powerapps`) — lists: `'RM Procurement Requests'` (central record), `'RM Procurement Line Items'` (one row per raw material per request), `'Raw Materials'` (raw-material catalog), `'RM User'` (roles), `'RM Procurement Approval Log'`, `'RM Procurement Execution Log'`, `Employee List`. Full column reference: [`docs/sharepoint-schema.md`](docs/sharepoint-schema.md).
- **Power Automate** flows called from Power Fx: `Parse_Invoice` / `Submit_Invoice` (AI invoice extraction), `Procurement_Notify_Receipt_Assignee`, `Procurement_Notify_Invoice_Provided`, `Procurement_Notify_Remind_Invoice`.

## Workflow

Requests move through the `Status` choice field on `'RM Procurement Requests'`:

```
New request (RequestFormScreen — must include ≥1 raw-material line item)
  │  Auto-skip to Executive if PurchaseAccordance = "Urgent"/"Unplanned"
  │  OR (EstimatedCost > 5000 AND cost-center-derived region = "AU")  → SkippedManagerReview
  ▼
Pending Manager   → "Approved (within budget)" → Pending Procurement
                  → otherwise                   → Pending Executive
Pending Executive → Reject → Rejected
                  → Approve / Approve with conditions → Pending Procurement
Pending Procurement → Reject → Rejected
                    → Proceed → Goods Receipt & Acceptance
Goods Receipt & Acceptance → Accepted → Pending Invoice (if invoice not yet submitted) or Pending Accounting
                           → Rejected → Rejected
                           → Requires Supplier Follow-up → Pending Supplier Follow-up
Pending Supplier Follow-up (Step 1: Requester, Step 2: Procurement, only if Adjustment)
                           → Pending Invoice (if invoice not yet submitted) or Pending Accounting
Pending Invoice → InvoiceSubmissionScreen / RequesterInvoiceScreen → Pending Accounting (or back to Pending Supplier Follow-up)
Pending Accounting → Completed   (the ONLY place Status is ever set to Completed)
```

## Roles

`Requester` · `Manager` · `Executive` · `Procurement` · `Accounting` · `Admin` — resolved from `'RM User'`; `HomeScreen` filters the request list per role (every role also sees their own submitted requests in addition to their role-specific queue). Users absent from `Employee List` are denied access.

## Editing / sync workflow

1. Edit the app in **Power Apps Studio**.
2. Export and unpack with the Power Platform CLI: `pac canvas unpack --msapp <file>.msapp --sources <this-repo>`.
3. The unpack regenerates `Controls/`, `References/`, etc. — these are git-ignored; commit only the `Src/` changes.

## Conventions

- All UI text, field names, comments, and code are **English**. SharePoint system column internal names are also English (site locale = en): `Title`, `Attachments`. The only remaining non-English is the `vi-VN` UserLocale.
- Status strings are repeated as literals across screens (no shared constant) — when changing the flow, update every screen that references the string.
