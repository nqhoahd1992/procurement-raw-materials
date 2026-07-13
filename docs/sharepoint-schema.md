# SharePoint Lists & Flows — Schema Reference

Authoritative data model for the Max Biocare Raw Materials Procurement app, reconstructed from `Src/*.pa.yaml`. Site: `maxbiocare.sharepoint.com/sites/Powerapps`.

> Only **custom / app-relevant columns** are listed. Every SharePoint list also carries the standard system columns (`ID`, `Created`, `Modified`, `Author`, `Editor`, `Attachments`, content-type, moderation, etc.) — omitted here for clarity. Update this file whenever a SharePoint column is added/renamed.

## Conventions

- **System columns use English internal names** (site locale = en) — use `Title` and `Attachments` directly in Power Fx, no quoting needed.
- **Choice** columns: write `{Value: "..."}`, read `Col.Value`.
- **Lookup / Person** columns: write `{Id: ..., Value: ...}`, read `Col.Value` / `Col.Id`. (These are SharePoint *lookup* columns pointing at another list — `Col.Value` resolves to the target's Title or ID per the `OdataQueryName` noted below.)
- **Required** columns are flagged ⚠ — a `Patch` that omits them fails.
- `RequestIDText` (plain-text copy of the request ID) is the delegable join key for log lookups: `LookUp('RM Procurement Execution Log', RequestIDText = Text(request.ID) && StepNumber = N)`.
- List names all carry an `RM ` prefix and contain spaces, so they're always single-quoted in Power Fx: `'RM Procurement Requests'`, `'RM Procurement Line Items'`, `'RM User'`, `'RM Procurement Approval Log'`, `'RM Procurement Execution Log'`.

---

## `'RM Procurement Requests'` — central request record

Required ⚠: `Status`, `RequesterEmail`, `ProcurementType`, `ProcurementDescription`, `PurchaseAccordance`, `EstimatedCost`, `Currency`, `RequiredDeliveryDate`, `DeliveryLocation`, `RequesterID`, `InvoiceRegion`, `CostCenter`.

Does **not** have a `Category`, `PreferredSupplier`, or `Department` column (all removed) — Category is now per raw-material (see `'Raw Materials'` below), Supplier no longer has a request-level "preferred" picker at all, and Department was dropped entirely since this app is now an internal Production-department-only workflow (no longer needs to record which department a request came from).

**Linking a request to a Project is optional.** `RequestFormScreen` has a `rdoHasProject` Radio ("Related to a Project?", `Items: =["No", "Yes"]` — defaults to "No", first item) gating the `ddProject_1` picker (`Project_List`, sourced directly — this app must have `Project_List` added as a data source in Studio, a separate Power Apps canvas app from `project-list`, same site). `ddProject_1` and its label/lookup-link are only `Visible` when `rdoHasProject.Selected.Value = "Yes"`, and submit only requires it in that case: `If(rdoHasProject.Selected.Value = "Yes" && IsBlank(ddProject_1.Selected), <error>, ...)`.

When `rdoHasProject = "Yes"` and a project is selected, `Cost Center` (`ddCostCenter_1`, an ordinary `ComboBox`) auto-fills via `DefaultSelectedItems: =Filter(Choices('RM Procurement Requests'.CostCenter), rdoHasProject.Selected.Value = "Yes" && Value = ddProject_1.Selected.CostCenter.Value)` and goes **read-only**: `DisplayMode: =If(Not(rdoHasProject.Selected.Value = "Yes"), DisplayMode.Edit, If(IsBlank(ddProject_1.Selected), DisplayMode.Disabled, DisplayMode.View))`. When `rdoHasProject = "No"` (or "Yes" but no project chosen yet), it stays a normal editable dropdown sourced from `Choices('RM Procurement Requests'.CostCenter)` — the user picks manually.

`Currency` is **never directly picked by the user** — it's a read-only `Label` (`lblCurrencyValue_1`, sits beside `txtEstimatedCost_1` in `colEstimatedCostRow`) computed purely from whichever `Cost Center` is currently set (whether auto-filled from a project or manually chosen): `Switch(ddCostCenter_1.Selected.Value, "Office Melbourne (Head Quarter)", "AUD", "Port Melbourne Warehouse", "AUD", "Max Biocare Research Park - Natural Inspirations@Yinnar", "AUD", "Max Biocare Research Park - Mar-Nuka Bay", "AUD", "Malay Warehouse", "MYR", "Singapore Warehouse", "SGD")`. Same warehouse-to-region mapping as `InvoiceRegion` below, just a currency code instead of a country code — kept as a separate duplicated `Switch` (this app's convention: no shared constants) since Patch also needs it standalone.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | Auto-built: `<employee> - <dd/mm/yyyy>` (no longer includes Category — see removal note above) |
| `Status` ⚠ | Choice | **Drives the workflow** — value list below |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `ProjectID` | Text | The related `project-list` project's business key (`Project_List.ProjectID`), set from `ddProject_1` when `rdoHasProject = "Yes"`; blank when the request isn't tied to a project |
| `ProcurementType` ⚠ | Choice | `Invoice Supplied`, `To be sourced by Procurement` |
| `InvoiceType` | Choice | `Official Invoice`, `Proforma Invoice`; blank unless `ProcurementType = "Invoice Supplied"` |
| `ProcurementDescription` ⚠ | Text (multiline) | |
| `PurchaseAccordance` ⚠ | Choice | incl. `Urgent`, `Unplanned` (auto-escalate to Executive) |
| `EstimatedCost` ⚠ | Number | |
| `Currency` ⚠ | Text | Confirmed Text (not Choice) via Studio's type-check — plain string values `AUD`/`MYR`/`SGD`. Never directly picked by the user; always computed from `Cost Center` via a `Switch` lookup (see note above) and written as a plain string on submit, no `{Value:}` wrapper |
| `BudgetReference` | Text | |
| `RequiredDeliveryDate` ⚠ | Date | |
| `DeliveryLocation` ⚠ | Choice | Independent from `CostCenter` below — delivery destination for goods, not the warehouse/office used for invoice filing |
| `CostCenter` ⚠ | Choice | 6 values: `Office Melbourne (Head Quarter)`, `Port Melbourne Warehouse`, `Max Biocare Research Park - Natural Inspirations@Yinnar`, `Max Biocare Research Park - Mar-Nuka Bay`, `Malay Warehouse`, `Singapore Warehouse`. `ddCostCenter_1` auto-fills/read-only from the selected Project's `CostCenter` when linked, otherwise a normal editable dropdown sourced from `Choices('RM Procurement Requests'.CostCenter)`. `InvoiceRegion` and `Currency` are both auto-derived from it via `Switch` lookups, never separately picked |
| `RequesterID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `ManagerApproverID` | Lookup→Employee List | blank when manager review skipped |
| `SkippedManagerReview` | Yes/No | true when Urgent/Unplanned **or** (EstimatedCost > 5000 AND cost-center-derived region = AU) |
| `InvoiceMode` | Choice | `Direct`, `Deferred`, `ViaRequester` — set by Procurement Execution |
| `InvoiceSubmitted` | Yes/No | true once the official invoice has been processed inline (drives whether Goods Receipt/Supplier Follow-up route to `Pending Invoice` or `Pending Accounting`) |
| `RequesterInvoiceURL` | Text (URL) | set when the Requester uploads the invoice (`ProcurementType = "Invoice Supplied"`) or re-uploads via `RequesterInvoiceScreen` |
| `OrderConfirmationURL` | Text (URL) | Deferred-invoice path, uploaded by Procurement |
| `RemittanceURL` | Text (URL) | Via-Requester path, uploaded by Procurement |
| `OfficialInvoiceLink` | Text (URL) | Final official invoice link, set by `Submit_Invoice` flow result |
| `ProcurementExecutedBy` | Lookup→Employee List | |
| `ProcurementExecutedAt` | DateTime | |
| `AccountingHandlerID` | Lookup→Employee List | chosen by Procurement on `InvoiceSubmissionScreen`/`ProcurementExecutionScreen` |
| `AccountingCompletedAt` | DateTime | set by `AccountingScreen` submit, alongside `Status = "Completed"` |
| `ConditionsText` | Text (multiline) | set on "Approve with conditions" |
| `PurchaseRequestLink` | Text (URL) | read-only in `RequestDetailScreen`; no `Patch` site found anywhere in the app — appears unwritten by current app code |
| `GRAssignedToID` | Lookup→Employee List | delegate for Goods Receipt; blank = Requester performs it |
| `GoodsReceiptBy` | Text | |
| `GoodsReceiptDate` | DateTime | |
| `GoodsReceiptStatus` | Choice | |
| `GoodsAcceptanceDecision` | Choice | `Accepted`, `Rejected`, `Requires Supplier Follow-up` |
| `GoodsReceiptRemarks` | Text (multiline) | |
| `GoodsReceiptAt` | DateTime | |
| `SFU1AssignedToID` | Lookup→Employee List | delegate for Supplier Follow-up Round 2 Step 1 (Requester); blank = Requester performs it |
| `FollowUpReceiptBy` | Text | |
| `FollowUpReceiptDate` | DateTime | |
| `FollowUpReceiptStatus` | Choice | same choices as `GoodsReceiptStatus` |
| `FollowUpAcceptanceDecision` | Choice | `Accepted`, `Accepted with Adjustment` |
| `CreditNote` | Text | required when follow-up decision = adjustment; cleared to `""` at the start of the Step-2 form |
| `Fulfillment` | Choice | `Fulfilled`, `Fulfilled with Adjustment` |
| `FollowUpRemarks` | Text (multiline) | |
| `FollowUpReceiptAt` | DateTime | |
| `SupplierFollowUpNotes` | Text (multiline) | |
| `FollowUpCompletedAt` | DateTime | |
| `InvoiceRegion` ⚠ | Choice | Country code (`AU`/`MY`/`SG`) used to file the invoice into the correct storage folder. **Not directly user-selected** — auto-derived on submit from `ddCostCenter_1.Selected.Value` via a `Switch` lookup table in `RequestFormScreen.pa.yaml` (warehouse/office → country) |
| `Attachments` | Attachments | invoice/supporting files; written via `Form1`+`SubmitForm` (Patch can't write attachments) |

### `Status` choice values and routing (exact strings — used as literals across all screens)

`Pending Manager`, `Pending Executive`, `Pending Procurement`, `Goods Receipt & Acceptance`, `Pending Supplier Follow-up`, `Pending Invoice`, `Pending Accounting`, `Completed`, `Rejected`.

Routing is **not** a straight line — see `CLAUDE.md`'s "The workflow" section for the full branching diagram. The two most important corrections vs. older assumptions:
- `"Completed"` is set in **exactly one place in the whole app**: `AccountingScreen`'s submit (`Patch(..., {Status: {Value: "Completed"}, ...})`). Neither `GoodsReceiptScreen` nor `SupplierFollowUpScreen` ever sets `Status` to `"Completed"` directly — both route to `"Pending Invoice"` or `"Pending Accounting"` depending on `InvoiceSubmitted`.
- `"Pending Invoice"` is a routing junction reachable from **both** Goods Receipt and Supplier Follow-up (not just once, right after Procurement Execution) — it represents "goods physically received, but official invoice paperwork not yet done", independent of `InvoiceMode`.

> Changing any string means updating every screen's filters, color maps, and `Patch`/`If`/`Switch` logic — there is no shared constant.

---

## `'RM Procurement Line Items'` — one row per raw material on a request

Required ⚠: none enforced by schema, but the app always writes `RequestID`, `RequestIDText`, `MaterialID`, `MaterialName`, `Unit`, `Quantity` together as a set on submit.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | Not set by the app's `Patch` — left blank/default |
| `RequestID` | Lookup→'RM Procurement Requests' | `{Id: wNewRequest.ID, Value: wNewRequest.Title}`, written once per line item via `ForAll` on `RequestFormScreen` submit |
| `RequestIDText` | Text | join key — `Filter('RM Procurement Line Items', RequestIDText = Text(gSelectedRequest.ID))` on every downstream screen |
| `MaterialID` | Lookup→'Raw Materials' | `{Id: MaterialID, Value: MaterialName}` |
| `MaterialName` | Text | copy of the raw material's `Title` at the time the line item was added |
| `Unit` | Text | one of `pcs`, `kg`, `box`, `set`, `liter`, `meter` (hardcoded list on `RequestFormScreen`, not a Choice column) |
| `Quantity` | Number | |
| `ReceivedQty1` | Number | Goods Receipt round-1 received quantity |
| `BatchNumber1` | Text | Goods Receipt round-1 batch number |
| `ExpiryDate1` | Date | Goods Receipt round-1 expiry date |
| `ReceivedQty2` | Number | Supplier Follow-up round-2 received quantity |
| `BatchNumber2` | Text | Supplier Follow-up round-2 batch number |
| `ExpiryDate2` | Date | Supplier Follow-up round-2 expiry date |

**How it's populated**: `RequestFormScreen` builds a working collection `colLineItems` (`{RowID, MaterialID, MaterialName, Unit, Quantity}`) via `galLineItems`/`btnAddLineItem`/`ddMaterial_1`/`ddUnit_1`/`txtQty_1`/`btnRemoveItem_1`. Submit requires ≥1 row, every row's `MaterialID <> 0`, non-blank `Unit`, and `Quantity > 0`. After the request `Patch` succeeds, a `ForAll(colLineItems, Patch('RM Procurement Line Items', Defaults(...), {...}))` writes one row per material, then `Clear(colLineItems)`.

**How it's read back**: `GoodsReceiptScreen`, `SupplierFollowUpScreen`, `RequestDetailScreen` each load `colLineItemsDetail` via `ClearCollect(colLineItemsDetail, Filter('RM Procurement Line Items', RequestIDText = Text(gSelectedRequest.ID)))` on `OnVisible`, then patch back the round-1/round-2 received-quantity/batch/expiry fields per row during Goods Receipt / Supplier Follow-up.

---

## `'Raw Materials'` — raw-material catalog

Only these columns are currently referenced anywhere in the app's Power Fx (confirmed by a repo-wide search); if more columns exist on the SharePoint list (e.g. INCI, CAS number) they are **not yet wired into the UI**:

| Column | Type | Notes |
|---|---|---|
| `ID` | Number (system) | used as `MaterialID` |
| `Tiêu đề` (Title) | Text | trade name — shown as the picker's display column (`ItemDisplayText: =ThisItem.Title`); copied into `RM Procurement Line Items.MaterialName` on add |
| `Code` | Text | shown read-only next to the material picker once a material is selected |
| `Category` | Text | shown read-only next to the material picker once a material is selected. **Not a Choice column** — read directly as `ddMaterial_1.Selected.Category`, no `.Value` |
| `Supplier` | Text | shown read-only next to the material picker once a material is selected — the raw material's own supplier, now the only place supplier information lives (the request-level `PreferredSupplier` field was removed) |

Loaded via `ClearCollect(colRawMaterials, 'Raw Materials')` on `RequestFormScreen.OnVisible`; `App.OnStart` only preloads `FirstN('Raw Materials', 1)` as a lightweight schema-shape seed before the user navigates anywhere.

---

## `'RM User'` — role assignment & app membership gate

Required ⚠: `Role`, `IsActive`, `EmployeeID`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | |
| `Role` ⚠ | Choice | `Requester`, `Manager`, `Executive`, `Procurement`, `Accounting`, `Admin` — **`Requester` must be added as an explicit Choice option on this SharePoint column** (List settings → `Role` column → edit choices). This is a SharePoint schema change, not something any `.pa.yaml` edit can do. |
| `IsActive` ⚠ | Yes/No | default true; manager picker filters `Role.Value="Manager" && IsActive` |
| `EmployeeID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `Note` | Text | |

Resolved in `App.OnStart` → `gCurrentUser` / `gUserRole` via `LookUp('RM User', EmployeeID.Id = gCurrentEmployee.ID)`. **This list is now the app's membership gate**: an employee must have a matching row here (any `Role`, including `Requester`) or `gCurrentUser` is blank and `HomeScreen` shows the "account not found" message instead of the app — being in `Employee List` alone is no longer enough (that was the old, tenant-wide behavior). There is no synthetic `Requester` fallback anymore; every requester needs a real `'RM User'` row.

Because of this gate, `GoodsReceiptScreen.ddAssignReceiver_GR` and `SupplierFollowUpScreen.ddAssignReceiver_SFU` (the "Assign to someone else" pickers) are filtered to only show `Employee List` rows that have an active `'RM User'` row: `Filter('Employee List', ID in ForAll(Filter('RM User', IsActive), EmployeeID.Id))`. This prevents assigning a Goods Receipt / Supplier Follow-up task to someone who wouldn't be able to log in to act on it.

---

## `'RM Procurement Approval Log'` — manager & executive decisions

Required ⚠: `RequestID`, `StepNumber`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | `Step <n> - <decision> - <request title>` |
| `RequestID` ⚠ | Lookup (→Title) | `{Id,Value}` |
| `StepNumber` ⚠ | Number | **`2` = Manager, `3` = Executive** (documented in the column itself) |
| `ApproverID` | Lookup→Employee List | |
| `Decision` | Choice | Manager: `Approved (within budget)`, `Needs clarification`, `Exceeds budget / unplanned` · Executive: `Reject`, `Approve with conditions`, `Approve` |
| `ApprovalConditions` | Text (multiline) | step 3, "Approve with conditions" |
| `RejectionReason` | Text (multiline) | step 3, "Reject" |
| `ManagerRemarks` | Text (multiline) | step 2 |
| `RequestIDText` | Text | join key |

---

## `'RM Procurement Execution Log'` — procurement / receipt / follow-up / invoice steps

Required ⚠: none.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | `Step <n> - <name> - <request title>` |
| `RequestID` | Lookup (→ID) | `{Id,Value}` |
| `RequestIDText` | Text | join key |
| `StepNumber` | Number | `1` Procurement Execution · `2` Accounting Handover · `3` Goods Receipt · `4` Supplier Follow-up (Requester) · `5` Supplier Follow-up (Procurement) · `6` Invoice Submission |
| `StepName` | Choice | matches the step |
| `ExecutedBy` | Lookup→Employee List | |
| `ExecutedAt` | DateTime | |
| `HandoverToID` | Lookup→Employee List | step 1 — accounting handler |
| `HandoverToIDText` | Text | step 1 |
| `SupplierSummary` | Text (multiline) | step 1 |
| `PurchaseOrderLink` | Text (URL) | step 1 |
| `Notes` | Text (multiline) | steps 1 (reject)/2/3 |
| `Attachments` | Attachments | Goods Receipt (step 3) and Supplier Follow-up round 2 (step 4) photo evidence — submitted via bound Forms (`frmGRLog_GR`, `frmSFU1Log_SFU`) since `Patch` alone can't write attachments |

The presence of step-4 / step-5 rows distinguishes the two stages of the supplier follow-up flow (`LookUp` by `StepNumber`); the presence of a step-6 row distinguishes whether `InvoiceSubmissionScreen` has already run for a request.

---

## `'Employee List'` — staff directory

Required ⚠: none.

| Column (internal) | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | employee display name |
| `Email` | Text | matched against `User().Email` in `App.OnStart` (`LookUp('Employee List', Email = User().Email)`). Elsewhere in the app the same column is read as `.email` (lowercase) — e.g. `LookUp('Employee List', ID = ...).email`, `ddAssignReceiver_GR.Selected.email`. Power Fx matches SharePoint column names case-insensitively so both forms work against the same column; keep both spellings in mind if this column is ever renamed |
| `Department` | Choice | **No longer read anywhere in the app** — `ddDepartment_1` and the request-level `Department` field were both removed (this app is now an internal Production-department-only workflow) |
| `JobTitle` | Text | |
| `City` | Text | |
| `Country` | Text | |
| `EmployeeCode` | Text | |

---

## Lists referenced by older versions of this app but no longer used

- **`Procurement_InvoiceData`** — zero references anywhere in current `Src/*.pa.yaml`. If it still exists on SharePoint it is orphaned; don't assume the app writes to it.
- **`Suppliers`** — zero references anywhere in current code. Removed together with the request-level `PreferredSupplier` field.

---

## Power Automate flows (Logic flows connector)

### `Parse_Invoice.Run(invoiceLink, requestId)` — AI invoice extraction

Called from both `ProcurementExecutionScreen` (Deferred and Via-Requester paths) and `InvoiceSubmissionScreen` (same two paths).

| Input | Type | |
|---|---|---|
| `text` | string | invoice link |
| `number` | number | request ID |

**Returns** (`gInvoiceResult`): `invoiceNumber`, `invoiceDate`, `totalAmount` (n), `taxAmount` (n), `supplierName`, `supplierABN`, `billedTo`, `currency`, `confidenceScore` (n), `jobId`, `attention`, `suggestedFilename`.

### `Submit_Invoice.Run(...)` — writes the official invoice

Called from `ProcurementExecutionScreen` (2 call sites — Deferred and Via-Requester) and `InvoiceSubmissionScreen` (2 call sites, same two paths). 16 positional args, identical order at all 4 call sites:

| # | Trigger param | App value |
|---|---|---|
| 1 | `text` | official invoice link |
| 2 | `number` | request ID |
| 3 | `text_1` | suggested filename (base) |
| 4 | `text_2` | invoice date |
| 5 | `text_3` | supplier name |
| 6 | `text_4` | billed-to |
| 7 | `text_5` | invoice number |
| 8 | `number_1` | total amount |
| 9 | `number_2` | tax amount |
| 10 | `text_6` | currency |
| 11 | `text_7` | AI jobId |
| 12 | `number_3` | AI confidenceScore |
| 13 | `text_8` | attention |
| 14 | `text_9` | **invoice Description field value** (all 4 call sites fill this — not a reserved/empty slot) |
| 15 | `text_10` | invoice region |
| 16 | `text_11` | ABN |

**Returns**: `newinvoicelink` (string).

### `Procurement_Notify_Receipt_Assignee.Run(assigneeEmail, assigneeName, requestTitle, requestId, notificationType, deliveryDate, category)`

Called from `GoodsReceiptScreen` (3 call sites) and `SupplierFollowUpScreen` (3 call sites) whenever the Goods Receipt / Supplier Follow-up assignee changes.

- `notificationType = "GoodsReceipt"` / `"SupplierFollowUp"` — new assignee notification (Outlook email + Teams Adaptive Card).
- `notificationType = "Unassigned"` — previous assignee no longer needs to act (email only).
- `category` — **always `""` at every call site in the current code.** This used to be the request-level Category field; Category now lives per raw-material and is no longer plumbed through to this flow.
- Connection: `app.admin@maxbiocare.com` pinned in "Run only users" — not invoker-provided.

### `Procurement_Notify_Invoice_Provided.Run(requestTitle, requestId, invoiceUrl)`

3 args. Called once, from `RequesterInvoiceScreen`, after a Requester successfully uploads/re-uploads a corrected invoice — notifies Procurement.

### `Procurement_Notify_Remind_Invoice.Run(requesterEmail, requesterName, requestTitle, requestIdText, rejectedByOrEmpty)`

5 args. Called twice from `InvoiceSubmissionScreen`:
- "Remind Requester" button — last arg `""`.
- "Request Re-upload" button — last arg = `gCurrentEmployee.Title` (the Procurement employee who rejected the submitted invoice).
