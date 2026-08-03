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

Required ⚠: `Status`, `RequesterEmail`, `ProcurementType`, `ProcurementDescription`, `PurchaseAccordance`, `EstimatedCost`, `Currency`, `RequiredDeliveryDate`, `DeliveryLocation`, `RequesterID`, `InvoiceRegion`, `CostCenter`, `ManagerApproverID`.

Does **not** have a `Category`, `PreferredSupplier`, or `Department` column (all removed) — Category is now per raw-material (see `'Raw Materials'` below), Supplier no longer has a request-level "preferred" picker at all, and Department was dropped entirely since this app is now an internal Production-department-only workflow (no longer needs to record which department a request came from).

**Linking a request to a Project is optional.** `RequestFormScreen` has a `rdoHasProject` Radio ("Related to a Project?", `Items: =["No", "Yes"]` — defaults to "No", first item) gating the `ddProject_1` picker (`Project_List`, sourced directly — this app must have `Project_List` added as a data source in Studio, a separate Power Apps canvas app from `project-list`, same site). `ddProject_1` and its label/lookup-link are only `Visible` when `rdoHasProject.Selected.Value = "Yes"`, and submit only requires it in that case: `If(rdoHasProject.Selected.Value = "Yes" && IsBlank(ddProject_1.Selected), <error>, ...)`.

`CostCenter` and `DeliveryLocation` are **no longer user-selectable** — both are hardcoded to `"Port Melbourne Warehouse"` on submit (shown as read-only labels `lblCostCenterValue_1`/`lblDeliveryLocationValue_1`), regardless of whether the request is linked to a Project. `InvoiceRegion` is likewise hardcoded to `"AU"` on submit. The old per-Project Cost Center auto-fill and the Cost-Center-driven region/currency `Switch` lookups no longer exist.

`Currency` is a manual dropdown (`ddCurrency_1`, `ModernCombobox`, sits beside `txtEstimatedCost_1` in `colEstimatedCostRow`) with exactly 2 options — `AUD` / `USD` — defaulting to `AUD`. It is no longer derived from Cost Center.

`RelatedSKU` is optional and **multi-value**: `cmbSKU` (`ModernCombobox`, `SelectMultiple: =true`, sits in its own row above the Raw Materials section) lets the requester link the request to one or more product SKUs, sourced from `Product_Database_SKU_Master` — a data source not otherwise used anywhere else in this app. Written on submit as `ForAll(cmbSKU.SelectedItems, {Id: ID, Value: Title})` (a table of `{Id,Value}`, matching a multi-value SharePoint Lookup column) — **not** `cmbSKU.Selected` (single-select shape). Empty table if nothing is selected.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | Auto-built: `<employee> - <dd/mm/yyyy>` (no longer includes Category — see removal note above) |
| `Status` ⚠ | Choice | **Drives the workflow** — value list below |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `ProjectID` | Text | The related `project-list` project's business key (`Project_List.ProjectID`), set from `ddProject_1` when `rdoHasProject = "Yes"`; blank when the request isn't tied to a project |
| `RelatedSKU` | Lookup→Product_Database_SKU_Master (multi) | Optional, multi-value; set from `cmbSKU` — see note above. Empty if nothing selected |
| `ProcurementType` ⚠ | Choice | `Invoice Supplied`, `To be sourced by Procurement` |
| `InvoiceType` | Choice | `Official Invoice`, `Proforma Invoice`; blank unless `ProcurementType = "Invoice Supplied"` |
| `ProcurementDescription` ⚠ | Text (multiline) | |
| `PurchaseAccordance` ⚠ | Choice | business classification (e.g. `Urgent`, `Unplanned`, `Normal`) — no longer affects routing; every request always goes through both Manager and Executive approval |
| `EstimatedCost` ⚠ | Number | |
| `Currency` ⚠ | Text | Confirmed Text (not Choice) via Studio's type-check. Manual dropdown (`ddCurrency_1`): `AUD` or `USD`, default `AUD` — no longer derived from Cost Center |
| `BudgetReference` | Text | |
| `RequiredDeliveryDate` ⚠ | Date | |
| `DeliveryLocation` ⚠ | Choice | Hardcoded to `"Port Melbourne Warehouse"` on submit (read-only label `lblDeliveryLocationValue_1`) — no longer user-selectable |
| `CostCenter` ⚠ | Choice | Hardcoded to `"Port Melbourne Warehouse"` on submit (read-only label `lblCostCenterValue_1`) — no longer user-selectable, no longer tied to the linked Project |
| `RequesterID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `ManagerApproverID` ⚠ | Lookup→Employee List | always set — every request requires a Manager Approver (no more skip-to-Executive path) |
| `SkippedManagerReview` | Yes/No | always written `false` on submit now — every request goes through both Manager and Executive approval. Kept only so older requests submitted before this change (where it may be `true`) still display correctly on `ExecutiveApprovalScreen`/`RequestDetailScreen` |
| `isExecutivePayment` | Yes/No | true when Executive approves an over-threshold request (`Currency <> "AUD" \|\| EstimatedCost > 10000`). Set by `ExecutiveApprovalScreen` on Approve/Approve-with-conditions; `Status` stays `"Pending Executive"` while this is true (UI shows "Pending Payment From Executive" as a computed label — see `CLAUDE.md`) |
| `InvoiceMode` | Choice | `Direct`, `Deferred`, `ViaRequester` — set by Procurement Execution |
| `InvoiceSubmitted` | Yes/No | true once the official invoice has been processed inline (drives whether Goods Receipt/Supplier Follow-up route to `Pending Invoice` or `Pending Accounting`) |
| `RequesterInvoiceURL` | Text (URL) | set when the Requester uploads the invoice (`ProcurementType = "Invoice Supplied"`) or re-uploads via `RequesterInvoiceScreen` |
| `OrderConfirmationURL` | Text (URL) | Deferred-invoice path, uploaded by Procurement |
| `RemittanceURL` | Text (URL) | Proof-of-payment attachment link. Two independent producers: `ExecutivePaymentScreen` (when `isExecutivePayment = true`) and `ProcurementExecutionScreen`'s own Path C "Remittance Advice Document" upload (`locIsViaRequester`, Procurement proceeding with a requester-supplied invoice). `ProcurementExecutionScreen` skips its own upload requirement and reuses this field's existing value when `isExecutivePayment = true` — see `CLAUDE.md` |
| `OfficialInvoiceLink` | Text (URL) | Final official invoice link, set by `Submit_Invoice` flow result |
| `ProcurementExecutedBy` | Lookup→Employee List | |
| `ProcurementExecutedAt` | DateTime | |
| `AccountingHandlerID` | Lookup→Employee List | **who actually completed the accounting step.** Written in exactly one place — `AccountingScreen`'s submit, as `gCurrentEmployee` — so it is blank until the request reaches `Completed`. It is *not* an assignment: there used to be an "Assign to Accounting Staff" picker on `ProcurementExecutionScreen` and `InvoiceSubmissionScreen` writing this field ahead of time, but the value was overwritten downstream anyway and never gated anything, so both pickers were removed. Read-only display on `RequestDetailScreen` ("Accounting Completed By") |
| `AccountingCompletedAt` | DateTime | set by `AccountingScreen` submit, alongside `Status = "Completed"` |
| `ConditionsText` | Text (multiline) | set on "Approve with conditions" |
| `PurchaseRequestLink` | Text (URL) | read-only in `RequestDetailScreen`; no `Patch` site found anywhere in the app — appears unwritten by current app code |
| `GRAssignedToID` | Lookup→Employee List | delegate for Goods Receipt round 1; blank = Requester performs it |
| `SFU1AssignedToID` | Lookup→Employee List | delegate for the **currently open** receipt round (round ≥ 2); blank = Requester performs it. **Cleared after every round** so the next round's receiver is picked again |
| `ReceiptRoundCount` ⬅ **new** | Number | how many receipt rounds have been submitted. `GoodsReceiptScreen` writes `1`; `SupplierFollowUpScreen` writes `N` each round. Used to number the next round (`+1`), stamp `RoundNumber` on the log and receipt-round rows, and label the UI |
| `LatestReceiptDecision` ⬅ **new** | Text | the acceptance decision of the most recent receipt round — `Accepted`, `Requires Supplier Follow-up`, or (round ≥ 2 only) `Accepted with Adjustment`. Plain Text (not Choice) so both the round-1 and round-N option sets fit. **Together with `Status` this is the whole state machine** — see the note below |
| `CreditNote` | Text | request-level, written **once** — only on the `Accepted with Adjustment` branch, by `ProcurementFollowUpScreen` |
| `Fulfillment` | Choice | `Fulfilled` (written by whichever screen closes with `Accepted`) or `Fulfilled with Adjustment` (written by `ProcurementFollowUpScreen`) |
| `InvoiceRegion` ⚠ | Choice | Country code used to file the invoice into the correct storage folder. **Not user-selected** — hardcoded to `"AU"` on submit, since `CostCenter` is now always `"Port Melbourne Warehouse"` |
| `Attachments` | Attachments | invoice/supporting files; written via `Form1`+`SubmitForm` (Patch can't write attachments) |

**All four receipt Choice columns were deleted — the dropdowns hold their options inline.** Nothing reads these columns' *value* on a request any more (the header moved per-round into `'RM Procurement Execution Log'`), so keeping them purely as `Choices()` sources was dead coupling. A literal `["a", "b"]` in Power Fx is a one-column table whose column is named `Value`, so every `.Selected.Value` reference kept working unchanged.

| Deleted column | Was `Items` of | Now |
|---|---|---|
| `GoodsReceiptStatus` | `ddReceiptStatus_GR` | `=["Fully Received", "Partially Received", "Incorrect Items", "Damaged Items"]` |
| `FollowUpReceiptStatus` | `ddReceiptStatus2_SFU` | same four values |
| `GoodsAcceptanceDecision` | `ddAcceptanceDecision_GR` | `=["Accepted", "Requires Supplier Follow-up"]` — **round 1 can no longer reject**; see the note below |
| `FollowUpAcceptanceDecision` | `ddAcceptanceDecision2_SFU` | `=["Requires Supplier Follow-up", "Accepted", "Accepted with Adjustment"]` — the first option is what lets the receiving loop continue past round 2; the third is what triggers the Credit Note |

**Goods Receipt round 1 no longer offers `Rejected`.** A short/damaged/incorrect delivery is now recorded as `Requires Supplier Follow-up` and handled through the receiving loop instead of terminating the request. The `Rejected` **`Status`** value is untouched and still reachable — `ManagerReviewScreen`, `ExecutiveApprovalScreen` and `ProcurementExecutionScreen` all still set it.

Trade-off of inlining: changing an option list is now a `.pa.yaml` edit pasted back into Studio, not a List-settings change on SharePoint — and because `LatestReceiptDecision` and the log's `AcceptanceDecision`/`ReceiptStatus` store these as plain text, renaming an option also means updating every `= "..."` comparison in the receiving loop.

**Deleted columns.** These held the round-1 / round-2 receipt header back when the app was capped at two rounds. There was no live receipt data at the time of the change, so they were **removed from the list** rather than kept as fallbacks — the app contains no read path for any of them:

| Deleted column | Replaced by |
|---|---|
| `GoodsReceiptBy` | `'RM Procurement Execution Log'.ReceivedBy` (Step 3) |
| `GoodsReceiptDate` | `…ReceiptDate` (Step 3) |
| `GoodsReceiptRemarks` | `…Notes` (Step 3) |
| `GoodsReceiptAt` | `…ExecutedAt` (Step 3) |
| `FollowUpReceiptBy` | `…ReceivedBy` (Step 4) |
| `FollowUpReceiptDate` | `…ReceiptDate` (Step 4) |
| `FollowUpRemarks` | `…Notes` (Step 4) |
| `FollowUpReceiptAt` | `…ExecutedAt` (Step 4) |
| `SupplierFollowUpNotes` | `…Notes` (Step 5, one row per round) |
| `FollowUpCompletedAt` | `…ExecutedAt` on the **final** Step-5 row. Deleted for a different reason from the rest: it was never per-round, just write-only — the app wrote it and no screen ever read it |

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
| `Title` (Title) | Text | Not set by the app's `Patch` — left blank/default |
| `RequestID` | Lookup→'RM Procurement Requests' | `{Id: wNewRequest.ID, Value: wNewRequest.Title}`, written once per line item via `ForAll` on `RequestFormScreen` submit |
| `RequestIDText` | Text | join key — `Filter('RM Procurement Line Items', RequestIDText = Text(gSelectedRequest.ID))` on every downstream screen |
| `MaterialID` | Lookup→'Raw Materials' | `{Id: MaterialID, Value: MaterialName}` |
| `MaterialName` | Text | copy of the raw material's `Title` at the time the line item was added |
| `Unit` | Text | one of `pcs`, `kg`, `box`, `set`, `liter`, `meter` (hardcoded list on `RequestFormScreen`, not a Choice column) |
| `Quantity` | Number | |

**Nothing about receiving lives on this list.** The 14 `…1` / `…2` columns it used to carry (`ReceivedQty1` `BatchNumber1` `ExpiryDate1` `QCNumber1` `RMPKCode1` `COALink1` `COA1Missing` and the matching `…2` set) were **deleted** when receipt rounds became unbounded — two fixed column sets can only ever hold two rounds. Per-round receipt data is now one row per (line item × round) in `'RM Procurement Receipt Rounds'`. No formula anywhere in the app reads the deleted columns.

**How it's populated**: `RequestFormScreen` builds a working collection `colLineItems` (`{RowID, MaterialID, MaterialName, Unit, Quantity}`) via `galLineItems`/`btnAddLineItem`/`ddMaterial_1`/`ddUnit_1`/`txtQty_1`/`btnRemoveItem_1`. Submit requires ≥1 row, every row's `MaterialID <> 0`, non-blank `Unit`, and `Quantity > 0`. After the request `Patch` succeeds, a `ForAll(colLineItems, Patch('RM Procurement Line Items', Defaults(...), {...}))` writes one row per material, then `Clear(colLineItems)`.

**How it's read back**: `GoodsReceiptScreen`, `SupplierFollowUpScreen`, `RequestDetailScreen` each load `colLineItemsDetail` via `ClearCollect(colLineItemsDetail, Filter('RM Procurement Line Items', RequestIDText = Text(gSelectedRequest.ID)))` on `OnVisible`. It is now a **read-only staging collection** — no screen patches receipt data back onto these rows any more. Each screen immediately projects it into a purpose-built collection:
- `GoodsReceiptScreen` / `SupplierFollowUpScreen` → `colRoundEntry`, the blank data-entry buffer for the round being recorded (`{LineItemID, MaterialID, MaterialName, Unit, OrderedQty, PrevReceivedQty, ReceivedQty, BatchNumber, ExpiryDate, QCNumber, RMPKCode, COALink}`). `PrevReceivedQty` = `Sum` of that line item's existing `colReceiptRounds` rows, so the receiver can see how much is still outstanding. On submit each row with `ReceivedQty > 0` becomes a new `'RM Procurement Receipt Rounds'` row.
- `RequestDetailScreen` → `colLineItemsSummary` (`{ID, MaterialID, MaterialName, Unit, Quantity, TotalReceivedQty}`), where `TotalReceivedQty` is the `Sum` across all rounds.

`colLineItemsDetail` does **not** carry `Category`/`Supplier` (those live on `'Raw Materials'`) — the screens re-look them up per row to display them:
- `RequestDetailScreen` (read-only, less crowded) shows them as full **Category**/**Supplier** columns alongside Trade Name/Unit/Qty, and clicking a row opens a per-round drill-down gallery over `Filter(colReceiptRounds, LineItemID = gSelectedLineItem.ID)`.
- `GoodsReceiptScreen`/`SupplierFollowUpScreen` (data-entry rows already packed with Received Qty/Batch/Expiry/QC Number/RM-PK Code/Link to COA inputs) instead show a small grey `"<Category> · <Supplier>"` subtitle line under the trade name, to avoid squeezing the input columns. Note these two screens project `MaterialID` into `colRoundEntry` as a **plain number** (`li.MaterialID.Id`), so the lookup there is `LookUp('Raw Materials', ID = ThisItem.MaterialID)` — no `.Id`.

`Src/COACompletionScreen.pa.yaml` is `gSelectedRequest`-scoped and reads into its own `colCOAOutstanding` from a single source — `'RM Procurement Receipt Rounds'` rows where `COAMissing` is true — patching back `COALink`/`COAMissing`. The receipt-round row `ID` is the unique key, so the input's `UpdateIf` keys off it directly; the earlier `RowKey`/`Source` disambiguation is gone, because a line item can now appear once per round and each of those is its own row. The screen does **not** touch `Status` or write an execution-log row — see `CLAUDE.md`'s "Link to COA completion" section.

**`COAMissing` exists purely to avoid a Power Apps delegation warning.** `Filter(..., IsBlank(COALink))` is flagged non-delegable by Studio — `IsBlank()` on a SharePoint text column can never be delegated by that connector, no formula rewrite fixes it. Every "is the COA link still missing" filter therefore reads the Yes/No column instead, which **is** delegable via a plain truthy check. The flag is written locally (evaluated against an already-fetched record inside a `Patch`, so `IsBlank` there is fine) at the same places the link itself is written: `GoodsReceiptScreen`'s `btnSubmit_GR` and `SupplierFollowUpScreen`'s `btnSubmitStep1_SFU` set it on the receipt-round rows they create, and `COACompletionScreen`'s save clears it.

Both `GoodsReceiptScreen` (`galLineItemsGR`) and `SupplierFollowUpScreen` (`galLineItems_SFU`) render one 56px row per material against `colRoundEntry`. `SupplierFollowUpScreen` additionally shows read-only **Ordered** and **Received So Far** columns before the input columns, so the receiver knows the outstanding balance for the round they are recording. `BatchNumber`/`ExpiryDate`/`QCNumber`/`RMPKCode` are required on submit for every row where `ReceivedQty > 0` — `btnSubmit_GR.OnSelect` / `btnSubmitStep1_SFU.OnSelect` validate each individually before writing. `COALink` is **not** validated as required — deliberately optional, see "Link to COA completion" in `CLAUDE.md`.

All of these labels set `Wrap: =false` — every raw-material line-items Gallery in this app uses a fixed `TemplateSize` per row. A long Category/Supplier value wrapping to a second line would get clipped against that fixed row height instead of growing it (standard Power Apps Gallery rows can't have a per-item variable height), so text is truncated to one line rather than allowed to wrap.

---

## `'Raw Materials'` — raw-material catalog

Only these columns are currently referenced anywhere in the app's Power Fx (confirmed by a repo-wide search); if more columns exist on the SharePoint list (e.g. INCI, CAS number) they are **not yet wired into the UI**:

| Column | Type | Notes |
|---|---|---|
| `ID` | Number (system) | used as `MaterialID` |
| `Title` (Title) | Text | trade name — shown as the picker's display column (`ItemDisplayText: =ThisItem.Title`); copied into `RM Procurement Line Items.MaterialName` on add |
| `Code` | Text | shown read-only next to the material picker once a material is selected |
| `Category` | Text | shown read-only next to the material picker once a material is selected. **Not a Choice column** — read directly as `ddMaterial_1.Selected.Category`, no `.Value` |
| `Supplier` | Text | shown read-only next to the material picker once a material is selected — the raw material's own supplier, now the only place supplier information lives (the request-level `PreferredSupplier` field was removed) |

Loaded via `ClearCollect(colRawMaterials, 'Raw Materials')` on `RequestFormScreen.OnVisible`; `App.OnStart` only preloads `FirstN('Raw Materials', 1)` as a lightweight schema-shape seed before the user navigates anywhere.

---

## `'RM User'` — role assignment & app membership gate

Required ⚠: `Role`, `IsActive`, `EmployeeID`.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | |
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
| `Title` (Title) | Text | `Step <n> - <decision> - <request title>` |
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
| `Title` (Title) | Text | `Step <n> - <name> - <request title>` |
| `RequestID` | Lookup (→ID) | `{Id,Value}` |
| `RequestIDText` | Text | join key |
| `StepNumber` | Number | `1` Procurement Execution · `2` Accounting Handover (written by `AccountingScreen`'s submit, i.e. at completion — despite the "handover" name it records the accounting step being *done*, not assigned) · `3` Goods Receipt **round 1** · `4` Goods Receipt **round N ≥ 2** · `5` Supplier Follow-up (Procurement, the `Accepted with Adjustment` close-out) · `6` Invoice Submission |
| `StepName` | Choice | matches the step |
| `RoundNumber` ⬅ **new** | Number | which receipt round this row belongs to. Step 3 always `1`; step 4 = `N`; step 5 = the round it follows up on. **Blank on rows written before the unlimited-rounds change** — every reader coalesces to `If(StepNumber = 3, 1, 2)` |
| `ReceivedBy` ⬅ **new** | Text | steps 3/4 — display name of whoever physically received the goods |
| `ReceiptDate` ⬅ **new** | Date | steps 3/4 |
| `ReceiptStatus` ⬅ **new** | Text | steps 3/4 — copy of the chosen `GoodsReceiptStatus` / `FollowUpReceiptStatus` choice value. Plain Text so both choice sets fit |
| `AcceptanceDecision` ⬅ **new** | Text | steps 3/4 — copy of the chosen acceptance decision. Plain Text, same reason |
| `ExecutedBy` | Lookup→Employee List | |
| `ExecutedAt` | DateTime | |
| `HandoverToID` | Lookup→Employee List | ⚠ **no longer written** — held the Accounting staff picked at step 1 / step 6. Both pickers were removed; safe to delete from the list |
| `HandoverToIDText` | Text | ⚠ **no longer written** — same; safe to delete |
| `SupplierSummary` | Text (multiline) | step 1 |
| `PurchaseOrderLink` | Text (URL) | step 1 |
| `Notes` | Text (multiline) | **shared column, meaning depends on `StepNumber`**: step 1 (reject reason) / 2 · steps 3 & 4 = that receipt round's Remarks · step 5 = that round's Procurement follow-up notes |
| `Attachments` | Attachments | receipt photos for the round — step 3 via `frmGRLog_GR`, step 4 via `frmSFU1Log_SFU`; `Patch` alone can't write attachments. One row per round ⇒ photos are naturally kept per round |

There can be **many step-4 rows per request** (one per receipt round from round 2 on) — never `LookUp` one expecting it to be unique. A step-5 row is written at most once, on the `Accepted with Adjustment` branch. Routing state lives on the request in `ReceiptRoundCount` / `LatestReceiptDecision`, not in the log. The presence of a step-6 row still distinguishes whether `InvoiceSubmissionScreen` has already run for a request.

---

## `'RM Procurement Receipt Rounds'` ⬅ **new list** — one row per (line item × receipt round)

Created for the unlimited-receipt-rounds change: the old fixed `…1` / `…2` columns on `'RM Procurement Line Items'` could only ever hold two rounds. Written by `GoodsReceiptScreen` (round 1) and `SupplierFollowUpScreen` (round N ≥ 2); **only materials with `ReceivedQty > 0` get a row**, so a round where a material wasn't delivered simply has no row for it.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | `R<n> - <material name>` |
| `RequestID` | Lookup→'RM Procurement Requests' | `{Id, Value}` |
| `RequestIDText` | Text | join key — every screen filters on this |
| `LineItemID` | Number | plain number copy of `'RM Procurement Line Items'.ID` (not a Lookup — kept delegable for equality filters and simple to `Sum`/`Filter` locally) |
| `RoundNumber` | Number | 1 = Goods Receipt, ≥2 = Supplier Follow-up rounds |
| `MaterialName` | Text | denormalized copy so COA/history screens don't have to re-join |
| `Unit` | Text | denormalized copy |
| `ReceivedQty` | Number | quantity received **in this round only** — cumulative received = `Sum` across rounds |
| `BatchNumber` | Text | required on submit |
| `ExpiryDate` | Date | required on submit |
| `QCNumber` | Text | required on submit |
| `RMPKCode` | Text | required on submit; defaults to the raw material's `Code` |
| `COALink` | Text (URL) | optional at submit — see "Link to COA completion" in `CLAUDE.md` |
| `COAMissing` | Yes/No | `IsBlank(COALink)` at write time; delegation-safe filter for the outstanding-COA views (see the delegation note under `'RM Procurement Line Items'`) |

`colReceiptRounds` mirrors this list per request and is the only source for "what has been received": `PrevReceivedQty` on the entry screens, `TotalReceivedQty` on `RequestDetailScreen`, the per-line-item drill-down gallery, and `ProcurementFollowUpScreen`'s "what was received in Round N" table all read it. This list replaced the deleted `…1`/`…2` line-item columns outright — there is no back-fill or synthetic-row path anywhere.

---

## `'Employee List'` — staff directory

Required ⚠: none.

| Column (internal) | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | employee display name |
| `Email` | Text | matched against `User().Email` in `App.OnStart` (`LookUp('Employee List', Email = User().Email)`). Elsewhere in the app the same column is read as `.email` (lowercase) — e.g. `LookUp('Employee List', ID = ...).email`, `ddAssignReceiver_GR.Selected.email`. Power Fx matches SharePoint column names case-insensitively so both forms work against the same column; keep both spellings in mind if this column is ever renamed |
| `Department` | Choice | **No longer read anywhere in the app** — `ddDepartment_1` and the request-level `Department` field were both removed (this app is now an internal Production-department-only workflow) |
| `JobTitle` | Text | |
| `City` | Text | |
| `Country` | Text | |
| `EmployeeCode` | Text | |

---

## Lists dropped by an older refactor (status notes)

- **`Procurement_InvoiceData`** — back in use as of the "Project Information" panel on `ManagerReviewScreen` / `ExecutiveApprovalScreen`, but **read-only**: those screens' `OnVisible` does `ClearCollect(colProjectInvoices, Filter(Procurement_InvoiceData, Not(IsBlank(gSelectedRequest.ProjectID)) && ProjectID = gSelectedRequest.ProjectID))` and the Actual Cost label sums `TotalAmount` grouped by `Currency`. Columns relied on: `ProjectID` (Text — added to the list on 2026-07-23, blank on older rows, which therefore match no project), `TotalAmount` (Number), `Currency` (Text). The app never writes to this list. Requires the list added as a connected data source in Studio.
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

Called from `ProcurementExecutionScreen` (2 call sites — Deferred and Via-Requester) and `InvoiceSubmissionScreen` (2 call sites, same two paths). 18 positional args, identical order at all 4 call sites:

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
| 17 | `text_12` | source app name — always the literal `"Raw Materials Procurement App"` |
| 18 | `text_13` | `gSelectedRequest.ProjectID` |

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
