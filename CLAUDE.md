# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Max Biocare Raw Materials Procurement** — a Power Apps Canvas App (DesktopOrTablet, 1366×768) that manages the end-to-end procurement request workflow for raw materials. The Power Fx source lives in `.pa.yaml` files.

This folder keeps **only the screen logic + UI** (`Src/`). The generated metadata that the full unpacked `.msapp` would contain (`Controls/`, `References/`, `Resources/`, `Properties.json`, `Header.json`, `AppCheckerResult.sarif`) has been **intentionally removed** — the workflow here is to read/edit Power Fx and paste it into **Power Apps Studio in the browser**, not to repack a `.msapp`. Because of that, this folder can no longer be packed with `pac canvas pack`; re-export from Power Apps if you ever need the full package again.

> Note: the sibling working directory `../ci_cd` is a **separate** Node.js/Docker project, unrelated to this Canvas App. Don't conflate the two.

## Layout

- `Src/*.pa.yaml` — one file per screen (the actual logic + UI). Screen order in `Src/_EditorState.pa.yaml`.
- `Src/App.pa.yaml` — `App.OnStart`: resolves the signed-in user and sets global state.

## Backend & connectors

All data lives in **SharePoint Online** (`maxbiocare.sharepoint.com/sites/Powerapps`). Lists:
- `'RM Procurement Requests'` — the central request record (status, approvers, costs, invoice, goods-receipt, follow-up fields). No longer carries `Category` or `PreferredSupplier` — both moved to the per-material line-item level (see `'Raw Materials'` below).
- `'RM Procurement Line Items'` — one row per raw material on a request (`MaterialID`/`MaterialName`, `Unit`, `Quantity`, plus Goods-Receipt/Supplier-Follow-Up round-1 and round-2 received-quantity/batch/expiry fields).
- `'Raw Materials'` — the raw-material catalog. Only `ID`, `Title` (trade name), `Code`, and `Category` are currently wired into the app; other catalog columns that may exist on SharePoint are not yet referenced anywhere in this code.
- `'RM User'` — maps an employee to a `Role` (Requester / Manager / Executive / Procurement / Accounting / Admin). **This list is now also the app's membership gate** — an employee must have a row here to use the app at all (see "Global state" and the membership note below). `Requester` must exist as an actual selectable value in this list's `Role` Choice column on SharePoint — that's a SharePoint list-settings change (List settings → `Role` column → edit choices), not something a `.pa.yaml` change can do. Because of this gate, the Goods Receipt / Supplier Follow-up "Assign to someone else" pickers (`ddAssignReceiver_GR`/`ddAssignReceiver_SFU`) are filtered to `Employee List` rows whose `ID` is among active `'RM User'` rows' `EmployeeID.Id` — an employee can only be assigned a receiving task if they can actually log into the app to perform it. Don't widen these pickers back to the full `Employee List` without re-adding some other way for the assignee to gain access.
- `'RM Procurement Approval Log'` — manager/executive decisions (StepNumber 2 = Manager, 3 = Executive).
- `'RM Procurement Execution Log'` — procurement/goods-receipt/follow-up/invoice step records (StepNumber 1 = Procurement Execution, 2 = Accounting Handover, 3 = Goods Receipt, 4 = Supplier Follow-up Requester, 5 = Supplier Follow-up Procurement, 6 = Invoice Submission).
- `'Employee List'`, `Project_List` (cross-app list from the sibling `project-list` app).
- `Product_Database_SKU_Master` — external product-SKU catalog, used only by `RequestFormScreen`'s optional multi-select `cmbSKU` picker (`SelectMultiple: =true`, writes `'RM Procurement Requests'.RelatedSKU` — a multi-value Lookup column — via `ForAll(cmbSKU.SelectedItems, {Id: ID, Value: Title})`). Not used anywhere else in this app.

Lists that **no longer exist in the app's code** (removed in a prior refactor — do not reintroduce without checking with the user first): `Procurement_InvoiceData`, `Suppliers`. If either still exists on SharePoint it is currently orphaned/unused by this app.

Full column-level schema (types, choice values, join keys): `docs/sharepoint-schema.md`.

**Power Automate** flows called from Power Fx:

Invoice flows (called from `ProcurementExecutionScreen` and `InvoiceSubmissionScreen`):
- `Parse_Invoice.Run(invoiceUrl, requestId)` — AI invoice extraction.
- `Submit_Invoice.Run(...)` — 17 positional args; writes the official invoice. Param 14 is the invoice **Description** field value (not a reserved/empty slot — see `docs/sharepoint-schema.md` for the full param table).

Assignment notification flow (called from `GoodsReceiptScreen` and `SupplierFollowUpScreen`):
- `Procurement_Notify_Receipt_Assignee.Run(assigneeEmail, assigneeName, requestTitle, requestId, notificationType, deliveryDate, category)`
  - `notificationType = "GoodsReceipt"` — notify new assignee they must perform Goods Receipt & Acceptance.
  - `notificationType = "SupplierFollowUp"` — notify new assignee they must perform Supplier Follow-up Round 2.
  - `notificationType = "Unassigned"` — notify previous assignee they no longer need to act (email only, no Adaptive Card).
  - Called on `btnSaveAssignment_GR` / `btnSaveAssignment_SFU` (assign new person) and `btnIWillReceive_GR` / `btnIWillReceive_SFU` (requester takes back the task).
  - Flow sends Outlook email + Teams Adaptive Card for assignment types; email only for Unassigned.
  - Connection: `app.admin@maxbiocare.com` pinned in "Run only users" — not invoker-provided.
  - **`category` is now always passed as `""`** at every call site — it was the request-level Category field, which no longer exists (Category now lives per raw-material). Treat this argument as vestigial; do not resurrect `gSelectedRequest.Category` here.

Invoice-reminder flows (called from `RequesterInvoiceScreen` and `InvoiceSubmissionScreen`):
- `Procurement_Notify_Invoice_Provided.Run(requestTitle, requestId, invoiceUrl)` — notifies Procurement after a Requester uploads a corrected invoice.
- `Procurement_Notify_Remind_Invoice.Run(requesterEmail, requesterName, requestTitle, requestIdText, rejectedByOrEmpty)` — "Remind Requester" (last arg `""`) and "Request Re-upload" (last arg = the rejecting employee's name) from `InvoiceSubmissionScreen`.

**Important:** SharePoint system column internal names are English (site locale = en) — use `Title` for the title field and `Attachments` for attachments. Custom columns are also English (`Status`, `EstimatedCost`, `ManagerApproverID`, etc.). `Employee List`'s email column is referenced as both `Email` (in `App.OnStart`'s `LookUp`) and `.email` (everywhere else) — Power Fx matches SharePoint column names case-insensitively, but keep this in mind if a rename ever breaks one usage and not the other.

## Global state (set in App.OnStart)

- `gCurrentEmployee` — row from `Employee List` matched by `User().Email`.
- `gCurrentUser` / `gUserRole` — row from `'RM User'` matched by `EmployeeID.Id = gCurrentEmployee.ID`. **Blank if the employee has no `'RM User'` row** — there is no more synthetic `"Requester"` fallback; a real `'RM User'` row (with `Role = "Requester"`) is required for ordinary requesters too, same as every other role.
- `gIsSpecialRole` — true for Manager/Executive/Procurement/Accounting/Admin (drives toolbar/filter visibility).
- `gSelectedRequest` — the request being viewed/acted on (set before `Navigate`).
- `gParsingInvoice`, `gHasInvoiceResult`, `gInvoiceResult`, `gShowRejectReason`.
- `gGRDelegationMode` / `gSFU1DelegationMode` — `"I will receive"` vs `"Assign to someone else"` toggle state for Goods Receipt / Supplier Follow-up.
- `gGRLogEntry` / `gGRPendingAttachments`, `gSFU1LogEntry` / `gSFU1PendingAttachments` — in-progress execution-log record + pending photo attachments for Goods Receipt / Supplier Follow-up before submit.
- `gSharePointAttachmentBase` — base URL used to resolve attachment links back to SharePoint.
- `colRawMaterials` — the `'Raw Materials'` catalog collection backing the line-item material picker; seeded with `FirstN('Raw Materials', 1)` in `App.OnStart` (schema placeholder) and fully reloaded (`ClearCollect(colRawMaterials, 'Raw Materials')`) on `RequestFormScreen.OnVisible`.
- `colLineItems` — the working collection of raw-material rows (`{RowID, MaterialID, MaterialName, Unit, Quantity}`) being built on `RequestFormScreen` before submit; persisted to `'RM Procurement Line Items'` via `ForAll` on successful submit, then cleared.
- `colLineItemsDetail` — per-request line items loaded from `'RM Procurement Line Items'` for viewing/receiving on GoodsReceiptScreen / SupplierFollowUpScreen / RequestDetailScreen / AccountingScreen-adjacent screens; carries the round-1/round-2 `ReceivedQty`/`BatchNumber`/`ExpiryDate` fields.
- `gStatusFilter` — set in `HomeScreen.OnVisible`, **not** in `App.OnStart`.

If `gCurrentEmployee.ID` **or** `gCurrentUser.ID` is blank, the user sees an "account not found" message and no UI (`HomeScreen`'s `lblNoRole` + every other `HomeScreen` container check `Not(IsBlank(gCurrentEmployee.ID)) && Not(IsBlank(gCurrentUser.ID))`). The app is membership-gated by **both** `Employee List` (must exist there) **and** `'RM User'` (must have an explicit role row there) — being in `Employee List` alone (the old behavior, tenant-wide) is no longer sufficient.

## The workflow — this is the core domain model

Requests move through a `Status` choice field. Each screen patches `'RM Procurement Requests'.Status` and writes a log row. The status string is also the value of the `HomeScreen` filter buttons and the gallery color coding.

```
RequestFormScreen (Requester submits, must add ≥1 raw-material line item, must pick a Manager Approver)
   │  Always Status = "Pending Manager", SkippedManagerReview = false (no more auto-skip-to-Executive path)
   ▼
Pending Manager ──(ManagerReviewScreen)──┐
   │ Any decision (Approved (within budget) / Needs clarification / Exceeds budget) → Pending Executive
   ▼
Pending Executive ──(ExecutiveApprovalScreen)──┐
   │ Reject → Rejected
   │ Approve / Approve with conditions → Pending Procurement (ConditionsText)
   ▼
Pending Procurement ──(ProcurementExecutionScreen)──┐
   │ Reject                      → Rejected
   │ Proceed                     → Goods Receipt & Acceptance
   │   (sets InvoiceMode: "Direct" | "Deferred" | "ViaRequester";
   │    InvoiceSubmitted = true only if the invoice was processed inline)
   ▼
Goods Receipt & Acceptance ──(GoodsReceiptScreen)──┐
   │ Accepted                    → Pending Invoice (if !InvoiceSubmitted) else Pending Accounting
   │ Rejected                    → Rejected
   │ Requires Supplier Follow-up → Pending Supplier Follow-up
   ▼
Pending Supplier Follow-up ──(SupplierFollowUpScreen, 2 steps)──┐
   │ Step 1 = Requester detail (ExecutionLog StepNumber 4)
   │   Accepted                  → Pending Invoice (if !InvoiceSubmitted) else Pending Accounting
   │   Accepted with Adjustment  → Pending Invoice/Pending Accounting (if !InvoiceSubmitted... else stays Pending Supplier Follow-up → Step 2)
   │ Step 2 = Procurement (ExecutionLog StepNumber 5, only if Step 1 = Adjustment; requires Credit Note)
   │   → Pending Invoice (if !InvoiceSubmitted) else Pending Accounting
   ▼
Pending Invoice ──(InvoiceSubmissionScreen [Procurement/Admin] or RequesterInvoiceScreen [Requester])──
   │ InvoiceSubmissionScreen submit (ExecutionLog StepNumber 6) →
   │   Pending Supplier Follow-up (if a Step-4 log exists but no Step-5 log yet)
   │   else Pending Accounting
   │ RequesterInvoiceScreen only uploads RequesterInvoiceURL + notifies Procurement — never changes Status
   ▼
Pending Accounting ──(AccountingScreen)──
   │ Submit → Completed   (this is the ONLY place Status is ever set to "Completed")
   ▼
Completed
```

Routing relies on these status strings being exact and consistent across `HomeScreen` (filters + gallery `Items` per-role filter), each action screen, and the `Switch`/`If` color maps. **When changing status names or the flow, update every screen that references the string** — there is no shared constant.

`SkippedManagerReview` on `'RM Procurement Requests'` and the `ExecutiveApprovalScreen` "Manager Review Skipped" banner (`rowManagerSkipped`, gated on `gSelectedRequest.SkippedManagerReview`) are now **write-only false going forward** — `RequestFormScreen` always writes `false` since every request goes through both approval levels. The field/banner are kept only so older requests submitted before this change (where it may be `true`) still display correctly; don't remove them.

`CostCenter` and `DeliveryLocation` on `RequestFormScreen` are no longer user-selectable — both are hardcoded to `"Port Melbourne Warehouse"` (read-only labels `lblCostCenterValue_1`/`lblDeliveryLocationValue_1`), and `InvoiceRegion` is hardcoded to `"AU"` accordingly. `Currency` is a manual `ddCurrency_1` dropdown (`AUD`/`USD`, default `AUD`) — no longer derived from Cost Center.

## Role-based visibility (HomeScreen)

The gallery `Items` filters `'RM Procurement Requests'` differently per `gUserRole` (always further filtered by `IsBlank(gStatusFilter) || Status.Value = gStatusFilter`):
- **Manager** → requests where `ManagerApproverID.Id = gCurrentEmployee.ID`, **or** their own requests (`RequesterEmail = User().Email`).
- **Procurement** → requests in status `"Pending Procurement"`, `"Pending Invoice"`, `"Pending Accounting"`, `"Goods Receipt & Acceptance"`, `"Pending Supplier Follow-up"`, `"Completed"`, `"Rejected"`, or their own requests. Does not include `"Pending Manager"`/`"Pending Executive"`.
- **Accounting** → a narrower version of Procurement's list: `"Pending Accounting"`, `"Goods Receipt & Acceptance"`, `"Pending Supplier Follow-up"`, `"Completed"`, or their own requests (no `"Pending Procurement"`, `"Pending Invoice"`, or `"Rejected"`).
- **Executive / Admin** → all requests, unfiltered.
- **Requester (default, and any unrecognized role)** → own requests (`RequesterEmail = User().Email`), plus requests where they're the current Goods Receipt or Supplier Follow-up assignee (`GRAssignedToID.Id = gCurrentEmployee.ID || SFU1AssignedToID.Id = gCurrentEmployee.ID`).

Filter buttons and "+ New Request" are **not** role-gated — every user who passes the membership gate (`Not(IsBlank(gCurrentEmployee.ID)) && Not(IsBlank(gCurrentUser.ID))`) sees the same filter bar and "+ New Request" button; only the underlying gallery `Items` differ per role. Keep the per-role `Items` filter in sync when adding new statuses, but don't assume the buttons themselves need per-role `Visible` logic — they currently don't have any.

## Conventions

- All UI text, field names, comments, and code are **English** (per global instruction). Vietnamese appears only as SharePoint system-column internal names and the `vi-VN` UserLocale.
- Patches use the `With({wPatched: Patch(...)}, If(IsBlank(wPatched.ID), Notify(error), ...success...))` pattern to guard against write failures — follow it for new writes.
- `RequestIDText` (text copy of the request ID) is used to look up log rows: `LookUp('RM Procurement Execution Log', RequestIDText = Text(gSelectedRequest.ID) && StepNumber = N)`.
- Colors are inline `RGBA(...)`; brand purple is `RGBA(83, 74, 183, 1)`.
