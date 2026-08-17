# SharePoint Lists & Flows — Schema Reference

Authoritative data model for the Max Biocare Raw Materials Procurement app, reconstructed from `Src/*.pa.yaml`. Site: `maxbiocare.sharepoint.com/sites/Powerapps`.

> Only **custom / app-relevant columns** are listed. Every SharePoint list also carries the standard system columns (`ID`, `Created`, `Modified`, `Author`, `Editor`, `Attachments`, content-type, moderation, etc.) — omitted here for clarity. Update this file whenever a SharePoint column is added/renamed.

## Conventions

- **System columns use English internal names** (site locale = en) — use `Title` and `Attachments` directly in Power Fx, no quoting needed.
- **Choice** columns: write `{Value: "..."}`, read `Col.Value`.
- **Lookup / Person** columns: write `{Id: ..., Value: ...}`, read `Col.Value` / `Col.Id`. (These are SharePoint *lookup* columns pointing at another list — `Col.Value` resolves to the target's Title or ID per the `OdataQueryName` noted below.)
- **Required** columns are flagged ⚠ — a `Patch` that omits them fails.
- `RequestIDText` (plain-text copy of the request ID) is the delegable join key for log lookups: `LookUp('RM Procurement Execution Log', RequestIDText = Text(request.ID) && StepNumber = N)`.
- **`RequestID` (the Lookup) is written everywhere but read nowhere in the app** — all 12 `Patch` sites write `RequestID` and `RequestIDText` together, and every `Filter`/`LookUp` in every screen keys off `RequestIDText`, because equality on a Lookup column isn't delegable in the SharePoint connector. Don't take that as licence to drop it: `'RM Procurement Approval Log'.RequestID` is Required ⚠ (an omitting `Patch` fails), the standalone `RM Procurement - Update COA Reminder for Procurement` flow reads `RequestID?['Id']`/`?['Value']` off `'RM Procurement Receipt Rounds'` to name each request without a second `Get items` (see `daily-coa-completion-reminder-flow.md`) — and losing it there degrades silently, with the run still reporting Succeeded — and the Lookup is what makes the row clickable through to its request when anyone opens these lists in SharePoint. It costs nothing to keep: it rides along in a `Patch` that is happening anyway.
- List names all carry an `RM ` prefix and contain spaces, so they're always single-quoted in Power Fx: `'RM Procurement Requests'`, `'RM Procurement Line Items'`, `'RM User'`, `'RM Procurement Approval Log'`, `'RM Procurement Execution Log'`.

---

## `'RM Procurement Requests'` — central request record

Required ⚠: `Status`, `RequesterEmail`, `ProcurementType`, `ProcurementDescription`, `PurchaseAccordance`, `EstimatedCost`, `Currency`, `RequiredDeliveryDate`, `DeliveryLocation`, `RequesterID`, `InvoiceRegion`, `CostCenter`, `ManagerApproverID`.

Does **not** have a `Category`, `PreferredSupplier`, or `Department` column (all removed) — Category is now per raw-material (see `'Raw Materials'` below), Supplier no longer has a request-level "preferred" picker at all, and Department was dropped entirely since this app is now an internal Production-department-only workflow (no longer needs to record which department a request came from).

**Linking a request to a Project is mandatory.** `RequestFormScreen` no longer has a "Related to a Project?" radio — that gate (`rdoHasProject`) was removed. `ddProject_1` (`Project_List`, sourced directly — this app must have `Project_List` added as a data source in Studio, a separate Power Apps canvas app from `project-list`, same site) is always visible, and `btnSubmit_1` rejects submit outright when `IsBlank(ddProject_1.Selected) || IsBlank(ddProject_1.Selected.ProjectID)`.

`CostCenter` and `DeliveryLocation` are **user-selectable `ModernCombobox` dropdowns** (`ddCostCenter_1`/`ddDeliveryLocation_1`), sourced live from the SharePoint Choice columns via `Choices('RM Procurement Requests'.CostCenter)` / `Choices('RM Procurement Requests'.DeliveryLocation)` — same pattern as `ddPurchaseAccordance_2`. Both default-select `"Port Melbourne Warehouse"` and are required at submit (folded into the combined "Please fill in all required fields" check). Adding a new option is a SharePoint List-settings change to the Choice column, not a `.pa.yaml` change. `InvoiceRegion` stays hardcoded to `"AU"` on submit — it isn't user-facing. The old per-Project Cost Center auto-fill and the Cost-Center-driven region/currency `Switch` lookups no longer exist.

`Currency` is a manual dropdown (`ddCurrency_1`, `ModernCombobox`, sits beside `txtEstimatedCost_1` in `colEstimatedCostRow`) with exactly 2 options — `AUD` / `USD` — defaulting to `AUD`. It is no longer derived from Cost Center.

`RelatedSKU` and `BudgetReference` are **no longer collected or displayed anywhere in this app** — the `cmbSKU` picker (multi-select, sourced from `Product_Database_SKU_Master`) and the `txtBudgetRef_1` input were removed from `RequestFormScreen`, and every read-only rendering of the two columns was removed along with them: `colBudgetReference`/`colRelatedSKU` on `RequestDetailScreen`, and `colReqBudgetRef_*`/`rowReqSKU_*` on `ProcurementExecutionScreen`/`InvoiceSubmissionScreen`. Neither column is Required ⚠, so no Patch/validation elsewhere needed to change. Both columns still exist on the list — older requests keep whatever values they already had — but nothing in `Src/` reads or writes them any more.

**This list now holds TWO kinds of row** — the Requester's request (parent) and the per-delivery batch Procurement creates from it (child). See "The two request types" below for what a child writes into every column. `RequestType` is the discriminator.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | Parent: `ID: <own ID> - <employee> - <dd/mm/yyyy> - <project Title>`, written by the **same second `Patch`** as `GroupKey` below, since the `ID` prefix isn't knowable until the row exists (the create `Patch` writes the same string minus that prefix). Child: `<parent Title> - #<DeliveryNumber>`, so a child inherits the parent's `ID:` prefix. **Not unique and not a key** — nothing prevents two children with the same number except the `DeliveryCount` allocation guard below |
| `RequestYear` ⬅ new | Number | 2 chữ số năm tạo request gốc (vd `26`). Delivery copy của cha. Chỉ dùng để cấp số và nhóm theo năm |
| `RequestSeq` ⬅ new | Number | Số thứ tự **trong năm đó**, chỉ cấp cho request gốc; delivery copy của cha. Cấp bằng `First(SortByColumns(Filter(…RequestType="Request" && RequestYear=…), "RequestSeq", Descending))+1` — mẫu delegable, không dùng `Max()` |
| `DisplayID` ⬅ new | Text | Mã hiển thị: `RMRQ-26-0001` (cha) · `RMRQ-26-0001-D1` (con). **Chỉ để người đọc** — không bao giờ dùng làm khoá tra cứu. `ID` thật vẫn là khoá duy nhất cho mọi `LookUp`/`Filter`/URL/tham số flow |
| `RequestType` ⬅ **new** | Choice | `Request` (parent) · `Delivery` (child). **Existing rows must be back-filled to `Request`** — a SharePoint column default applies only to new items, and `RequestType.Value = "Request"` does not match blank, so an un-backfilled row disappears from `HomeScreen` for every role. Do **not** use `IsBlank(ParentRequestID)` as a substitute discriminator: `IsBlank()` on a Number is not delegable (same reason `COAMissing` exists) |
| `ParentRequestID` ⬅ **new** | Number | Blank/0 on a parent; the parent's `ID` on a child. Plain **Number**, not a Lookup, so `=` delegates |
| `DeliveryNumber` ⬅ **new** | Number | `0` on a parent, `1..n` on a child. **Named `DeliveryNumber`, not `BatchNumber`** — `'RM Procurement Receipt Rounds'.BatchNumber` already means the *material's lot number*, and reusing the word across the two lists made every formula ambiguous |
| `DeliveryCount` ⬅ **new** | Number | On the **parent** — how many deliveries have been created. `DeliveryBatchFormScreen` allocates the next number as `DeliveryCount + 1` and re-reads the parent to abort if it changed; `Max(DeliveryNumber)` over the siblings would still race |
| `GroupKey` ⬅ **new** | Number | = own `ID` on a parent, = parent's `ID` on a child. **Sort key only.** `HomeScreen` sorts `GroupKey` desc then `DeliveryNumber` asc, which is what puts each parent directly above its own deliveries in one flat delegable gallery. Written by a **second** `Patch` on create, because `ID` isn't known until the row exists — a parent missing `GroupKey` sinks silently to the bottom of the list |
| `ClosedByID` ⬅ **new** | Lookup→Employee List | who closed the parent |
| `ClosedAt` ⬅ **new** | DateTime | when the parent was closed |
| `CloseRemarks` ⬅ **new** | Text (multiline) | required when closing — the written reason, which matters most when the supplier under-delivered and the shortfall is being written off |
| `Status` ⚠ | Choice | **Drives the workflow** — value list below. The parent's and the child's status sets are **disjoint** |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `ProjectID` ⚠ (in practice) | Text | The related `project-list` project's business key (`Project_List.ProjectID`), set from `ddProject_1`. Selecting a Project is now mandatory on `RequestFormScreen` — not a SharePoint Required ⚠ column, but `btnSubmit_1` blocks submit if blank |
| `RelatedSKU` | Lookup→Product_Database_SKU_Master (multi) | **No longer written or displayed anywhere in this app** — the `cmbSKU` picker was removed from `RequestFormScreen`, and its read-only renderings on `RequestDetailScreen`/`ProcurementExecutionScreen`/`InvoiceSubmissionScreen` were removed along with it. Always blank on a request submitted after this change |
| `ProcurementType` ⚠ | Choice | `Invoice Supplied`, `To be sourced by Procurement` |
| `InvoiceType` | Choice | `Official Invoice`, `Proforma Invoice`; blank unless `ProcurementType = "Invoice Supplied"` |
| `ProcurementDescription` ⚠ | Text (multiline) | |
| `PurchaseAccordance` ⚠ | Choice | business classification (e.g. `Urgent`, `Unplanned`, `Normal`) — no longer affects routing; every request always goes through both Manager and Executive approval |
| `EstimatedCost` ⚠ | Number | |
| `Currency` ⚠ | Text | Confirmed Text (not Choice) via Studio's type-check. Manual dropdown (`ddCurrency_1`): `AUD` or `USD`, default `AUD` — no longer derived from Cost Center |
| `BudgetReference` | Number | **No longer written or displayed anywhere in this app** — `txtBudgetRef_1` was removed from `RequestFormScreen`, and its read-only renderings on `RequestDetailScreen`/`ProcurementExecutionScreen`/`InvoiceSubmissionScreen` were removed along with it. Always blank on a request submitted after this change |
| `RequiredDeliveryDate` ⚠ | Date | |
| `DeliveryLocation` ⚠ | Choice | User-selectable `ModernCombobox` (`ddDeliveryLocation_1`), `Items: =Choices('RM Procurement Requests'.DeliveryLocation)`, default-selected `"Port Melbourne Warehouse"` |
| `CostCenter` ⚠ | Choice | User-selectable `ModernCombobox` (`ddCostCenter_1`), `Items: =Choices('RM Procurement Requests'.CostCenter)`, default-selected `"Port Melbourne Warehouse"` — no longer tied to the linked Project |
| `RequesterID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `ManagerApproverID` ⚠ | Lookup→Employee List | always set — every request requires a Manager Approver (no more skip-to-Executive path) |
| `SkippedManagerReview` | Yes/No | **dead — no longer written or read by the app.** Every request goes through both Manager and Executive approval, so nothing escalates past a level. Pre-change rows may still hold `true`; the column is kept on the list for that history only |
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
| `GRAssignedToID` | Lookup→Employee List | delegate for Goods Receipt round 1; blank = Requester performs it |
| `SFU1AssignedToID` | Lookup→Employee List | delegate for the **currently open** receipt round (round ≥ 2); blank = Requester performs it. **Cleared after every round** so the next round's receiver is picked again |
| `ReceiptRoundCount` ⬅ **new** | Number | how many receipt rounds have been submitted. `GoodsReceiptScreen` writes `1`; `SupplierFollowUpScreen` writes `N` each round. Used to number the next round (`+1`), stamp `RoundNumber` on the log and receipt-round rows, and label the UI |
| `LatestReceiptDecision` ⬅ **new** | Text | the acceptance decision of the most recent receipt round — `Accepted`, `Requires Supplier Follow-up`, or (round ≥ 2 only) `Accepted with Adjustment`. Plain Text (not Choice) so both the round-1 and round-N option sets fit. **Together with `Status` this is the whole state machine** — see the note below |
| `CreditNote` | Text | request-level, written **once** — only on the `Accepted with Adjustment` branch, by `ProcurementFollowUpScreen` |
| `Fulfillment` | Choice | `Fulfilled` (written by whichever screen closes with `Accepted`) or `Fulfilled with Adjustment` (written by `ProcurementFollowUpScreen`) |
| `InvoiceRegion` ⚠ | Choice | Country code used to file the invoice into the correct storage folder. **Not user-selected** — hardcoded to `"AU"` on submit regardless of the chosen `CostCenter`/`DeliveryLocation` |
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

### The two request types

Both live in `'RM Procurement Requests'`; `RequestType` tells them apart.

| | `Request` (parent) | `Delivery` (child) |
|---|---|---|
| Created by | Requester, `RequestFormScreen` | Procurement, `DeliveryBatchFormScreen` |
| Line items | all materials requested | only the materials in this delivery, with that delivery's quantity, each carrying `ParentLineItemID` |
| Invoice columns | **unused** — the parent never carries an invoice any more | its own `InvoiceMode` / `InvoiceSubmitted` / `OfficialInvoiceLink`. **This is what makes one request able to have many invoices** |
| Receipt rounds | none of its own; rolls up its children's via `ParentRequestID` | its own, `RequestIDText` = the child's ID |
| Legal `Status` | `Pending Manager`, `Pending Executive`, `Pending Procurement`, `In Delivery`, `Delivered`, `Rejected` | `Goods Receipt`, `Supplier Follow-up`, `Pending Invoice`, `Pending Accounting`, `Completed`, `Cancelled` |

Because the two sets are disjoint, every existing `Status` gate on the downstream screens is **already type-safe** and needed no `RequestType` check added. `RequestType` is only consulted by `HomeScreen`'s Manager branch and by presentation logic.

**What a child writes into each Required ⚠ column** (a `Patch` omitting any of them fails, surfacing only as a generic "Failed to create the delivery"):

| Column | Child's value | Why it matters |
|---|---|---|
| `RequesterEmail` ⚠ | copy of parent | **Load-bearing** — `HomeScreen`'s default branch, `btnActionGoodsReceipt.Visible` and `btnSubmit_GR`'s guard all key on it |
| `ManagerApproverID` ⚠ | copy of parent | Required, so unavoidable. It is why `HomeScreen`'s Manager branch adds `&& RequestType.Value = "Request"` — otherwise every manager sees every delivery of every request they ever approved |
| `EstimatedCost` ⚠ | **`0`** | Copying the parent's figure would multiply any `Sum(EstimatedCost)` by (1 + deliveries). Nothing sums it today, so `0` costs nothing and keeps the table additive-safe. `HomeScreen`'s meta line already hides a `0` cost |
| `RequiredDeliveryDate` ⚠ | **this delivery's own date**, from the form | It is what `ProcurementNotify` arg 6 prints in the receiver's email and what the goods-receipt reminder flow prints. Copying the parent's date is a silently wrong email |
| `ProcurementDescription` ⚠ | `"Delivery batch #n of request #N"` + optional notes | **Never a verbatim copy** — that would make parent and child indistinguishable in Quick View and in every SharePoint list view |
| `InvoiceRegion` ⚠ | copy of parent | `Submit_Invoice` param 15 is read off the row being invoiced; blank here files the invoice into the wrong folder |
| `RequesterID` ⚠, `ProcurementType` ⚠, `PurchaseAccordance` ⚠, `Currency` ⚠ | copy of parent | display + valid-choice requirements |
| `DeliveryLocation` ⚠, `CostCenter` ⚠ | `"Port Melbourne Warehouse"` | hardcoded everywhere |

Non-required but deliberate on a child:

| Column | Child's value | Why |
|---|---|---|
| `ProjectID` | **copy of parent — mandatory in practice** | `Submit_Invoice` resolves the project itself from the request ID at param 2, which is now the *child's* ID. Blank here ⇒ `Procurement_InvoiceData.ProjectID` blank ⇒ the Manager/Executive Actual-Cost panel silently under-reports, with the flow run still Succeeded. **Verified against the live flow — see "What `Submit_Invoice` actually reads" below: `ProjectID` is the only request column it touches** |
| `RequirementFiles` | **never copied** | `colRequirementFiles` composes URLs as `gSharePointAttachmentBase & Text(<request ID>) & …` and the files are attached to the **parent's** item. A copied name-list would 404 on three screens. `RequestDetailScreen` instead reads the parent's list and builds the URLs from `ParentRequestID` |
| `isExecutivePayment` | **never copied** | copying `true` would put a "Process Payment →" button on a delivery |
| `RelatedSKU` | skipped | avoids a second multi-value Lookup write that nothing reads |

### `Status` choice values and routing (exact strings — used as literals across all screens)

`Pending Manager`, `Pending Executive`, `Pending Procurement`, **`In Delivery`**, **`Delivered`**, `Goods Receipt`, `Supplier Follow-up`, `Pending Invoice`, `Pending Accounting`, `Completed`, **`Cancelled`**, `Rejected`.

The three new values:
- **`In Delivery`** (parent) — set by `ProcurementExecutionScreen` on Proceed. The parent's lifecycle ends here; it is now an open container that Procurement adds deliveries to. **This breaks the invariant both reminder flows rest on** ("being in a status means the work on *this row* is incomplete") — an `In Delivery` parent is waiting on its children, not on anyone acting on itself. `Delivered` is what drains it.
- **`Delivered`** (parent, terminal) — set manually by Procurement/Admin via `RequestDetailScreen`'s Close Request dialog. Requires `CloseRemarks`, and is blocked while any child is neither `Completed` nor `Cancelled`. **Deliberately allowed when the delivered quantity is short of ordered** — the supplier may never deliver in full, so a short close is a normal outcome that just needs a written reason. There is no auto-close.
- **`Cancelled`** (child, terminal) — a delivery created in error. Blocked once `ReceiptRoundCount > 0`. Without it a mistaken delivery would be immortal (round 1 deliberately has no `Rejected`, and a child passes no approval screen), and it would block the parent from ever being closable.

`Rejected` on a parent is reachable from exactly two places, both terminal: Executive Reject (`ExecutiveApprovalScreen`) and Procurement Reject (`ProcurementExecutionScreen`). A request rejected at Procurement never has any delivery — which is why the "+ Add Delivery" button gates on `Status = "In Delivery"` rather than on "has passed Procurement Execution".

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
| `RequestIDText` | Text | join key — `Filter('RM Procurement Line Items', RequestIDText = Text(gSelectedRequest.ID))` on every downstream screen. **May point at either a parent or a delivery** (see below) |
| `ParentLineItemID` ⬅ **new** | Number | `0` on a parent's rows; on a delivery's rows, the `ID` of the parent line item it draws down. This is what lets the parent roll up "delivered so far per material" across all its deliveries |
| `MaterialCode` ⬅ new | Text | the raw material's `Code`, denormalised onto the line item at creation. Read as `RMPKCode` by every screen's `colLineItemsDetail` projection. Exists so no screen has to `LookUp('Raw Materials', …)` inside a `ForAll` — that is client-side per row and always raises a delegation warning |
| `MaterialID` | Lookup→'Raw Materials' | `{Id: MaterialID, Value: MaterialName}` |
| `MaterialName` | Text | copy of the raw material's `Title` at the time the line item was added |
| `Unit` | Text | one of `pcs`, `kg`, `box`, `set`, `liter`, `meter` (hardcoded list on `RequestFormScreen`, not a Choice column) |
| `Quantity` | Number | |

**This list now holds rows for both request types.** A parent's rows are what the Requester asked for (`ParentLineItemID = 0`); a delivery's rows are the subset Procurement says is arriving in that delivery, with that delivery's quantity. **This is the fix for the original complaint** — `GoodsReceiptScreen` still does the same unchanged `Filter(…, RequestIDText = Text(gSelectedRequest.ID))`, but because it now runs against a delivery it naturally lists only that delivery's materials instead of every material on the request. `Quantity` on a delivery row therefore means *expected in this delivery*, not *ordered*, and `colRoundEntry.OrderedQty` (and the "Ordered" column on both receipt screens) inherits that meaning.

**Nothing about receiving lives on this list.** The 14 `…1` / `…2` columns it used to carry (`ReceivedQty1` `BatchNumber1` `ExpiryDate1` `QCNumber1` `RMPKCode1` `COALink1` `COA1Missing` and the matching `…2` set) were **deleted** when receipt rounds became unbounded — two fixed column sets can only ever hold two rounds. Per-round receipt data is now one row per (line item × round) in `'RM Procurement Receipt Rounds'`. No formula anywhere in the app reads the deleted columns.

**How it's populated**: `RequestFormScreen` builds a working collection `colLineItems` (`{RowID, MaterialID, MaterialName, Unit, Quantity}`) via `galLineItems`/`btnAddLineItem`/`ddMaterial_1`/`ddUnit_1`/`txtQty_1`/`btnRemoveItem_1`. Submit requires ≥1 row, every row's `MaterialID <> 0`, non-blank `Unit`, and `Quantity > 0`. After the request `Patch` succeeds, a `ForAll(colLineItems, Patch('RM Procurement Line Items', Defaults(...), {...}))` writes one row per material, then `Clear(colLineItems)`.

**How it's read back**: `GoodsReceiptScreen`, `SupplierFollowUpScreen`, `RequestDetailScreen` each load `colLineItemsDetail` via `ClearCollect(colLineItemsDetail, Filter('RM Procurement Line Items', RequestIDText = Text(gSelectedRequest.ID)))` on `OnVisible`. It is now a **read-only staging collection** — no screen patches receipt data back onto these rows any more. Each screen immediately projects it into a purpose-built collection:
- `GoodsReceiptScreen` / `SupplierFollowUpScreen` → `colRoundEntry`, the blank data-entry buffer for the round being recorded (`{LineItemID, MaterialID, MaterialName, Unit, OrderedQty, PrevReceivedQty, ReceivedQty, BatchNumber, ExpiryDate, QCNumber, RMPKCode, COALink}`). `PrevReceivedQty` = `Sum` of that line item's existing `colReceiptRounds` rows, so the receiver can see how much is still outstanding. On submit each row with `ReceivedQty > 0` becomes a new `'RM Procurement Receipt Rounds'` row.
- `RequestDetailScreen` → `colLineItemsSummary` (`{ID, MaterialID, MaterialName, Unit, Quantity, TotalReceivedQty}`), where `TotalReceivedQty` is the `Sum` across all rounds.

`colLineItemsDetail` does **not** carry `Category`/`Supplier` (those live on `'Raw Materials'`) — the screens re-look them up per row to display them:
- `RequestDetailScreen` (read-only) shows them as a grey `"<Code> · <Category> · <Supplier>"` subtitle under the trade name in its flat line-items table. Rows are **not** selectable — the per-material drill-down panel was removed; per-round receipt detail is read from the "View receipt details" dialog on each round of the Goods Receipt history instead (`colRoundDetails`, see `CLAUDE.md`).
- `GoodsReceiptScreen`/`SupplierFollowUpScreen` (data-entry rows already packed with Received Qty/Batch/Expiry/QC Number/Link to COA inputs, plus a read-only RM Code cell) instead show a small grey `"<Category> · <Supplier>"` subtitle line under the trade name, to avoid squeezing the input columns. Note these two screens project `MaterialID` into `colRoundEntry` as a **plain number** (`li.MaterialID.Id`), so the lookup there is `LookUp('Raw Materials', ID = ThisItem.MaterialID)` — no `.Id`.

`Src/ReceiptAmendmentScreen.pa.yaml` is `gSelectedRequest`-scoped and reads **every** one of that request's `'RM Procurement Receipt Rounds'` rows into `colReceiptAmend` — not just the ones missing a COA — patching back `BatchNumber`/`ExpiryDate`/`QCNumber`/`COALink` plus a recomputed `COAMissing` (`ReceivedQty` is displayed read-only — see `CLAUDE.md` for why it is not amendable). Each field is held twice in the collection (`X` = stored, `NewX` = being edited) so the save can `Filter` down to rows that actually changed. The receipt-round row `ID` is the unique key, so both the inputs' `UpdateIf` and the save's `Patch` key off it directly; the earlier `RowKey`/`Source` disambiguation is gone, because a line item now yields one row per round rather than two fixed column sets. It replaces the former `COACompletionScreen`/`colCOAOutstanding` pair, which could only fill a blank `COALink`.

**`COAMissing` exists purely to avoid a Power Apps delegation warning.** `Filter(..., IsBlank(COALink))` is flagged non-delegable by Studio — `IsBlank()` on a SharePoint text column can never be delegated by that connector, no formula rewrite fixes it. Every "is the COA link still missing" filter therefore reads the Yes/No column instead, which **is** delegable via a plain truthy check. The flag is written locally (evaluated against an already-fetched record inside a `Patch`, so `IsBlank` there is fine) at the same places the link itself is written: `GoodsReceiptScreen`'s `btnSubmit_GR` and `SupplierFollowUpScreen`'s `btnSubmitStep1_SFU` set it on the receipt-round rows they create, and `COACompletionScreen`'s save clears it.

Both `GoodsReceiptScreen` (`galLineItemsGR`) and `SupplierFollowUpScreen` (`galLineItems_SFU`) render one 56px row per material against `colRoundEntry`. `SupplierFollowUpScreen` additionally shows read-only **Ordered** and **Received So Far** columns before the input columns, so the receiver knows the outstanding balance for the round they are recording. `BatchNumber`/`ExpiryDate`/`QCNumber`/`COALink` are required on submit for every row where `ReceivedQty > 0` — `btnSubmit_GR.OnSelect` / `btnSubmitStep1_SFU.OnSelect` validate each individually before writing. **`RMPKCode` is not validated and has no input control**: `lblGRItemRMCode` / the `_SFU` twin are Labels, and the value is seeded in `OnVisible` from `LookUp('Raw Materials', ID = wMatID).Code`. The receiver cannot type it, so there would be nothing for a guard to prompt them to fix.

All of these labels set `Wrap: =false` — every raw-material line-items Gallery in this app uses a fixed `TemplateSize` per row. A long Category/Supplier value wrapping to a second line would get clipped against that fixed row height instead of growing it (standard Power Apps Gallery rows can't have a per-item variable height), so text is truncated to one line rather than allowed to wrap.

---

## `'Raw Materials'` — raw-material catalog

Only these columns are currently referenced anywhere in the app's Power Fx (confirmed by a repo-wide search); if more columns exist on the SharePoint list (e.g. INCI, CAS number) they are **not yet wired into the UI**:

| Column | Type | Notes |
|---|---|---|
| `ID` | Number (system) | used as `MaterialID` |
| `Title` (Title) | Text | trade name — the material picker's primary display field; copied into `RM Procurement Line Items.MaterialName` on add |
| `Code` | Text | the picker's secondary display field, and searchable alongside `Title`; also shown read-only next to the picker once a material is selected |
| `Category` | **Choice** | shown read-only next to the material picker once a material is selected. Being a Choice, **every read needs `.Value`** — `ddMaterial_1.Selected.Category.Value`, `wMat.Category.Value`. Adding a category is a List settings change, not a code change. It was a Text column until Aug 2026; if a formula ever concatenates `wMat.Category` without `.Value` the whole label errors, since a Choice record has no string coercion |
| `Supplier` | **Lookup→`RM Suppliers`** (shows `Title`) | shown read-only next to the material picker once a material is selected — the raw material's own supplier, the only place supplier information lives (the request-level `PreferredSupplier` field was removed). Being a Lookup, **every read needs `.Value`** |
| `SupplierID` | **Lookup→`RM Suppliers`** (shows `Code`) | the same supplier by code. **Not referenced anywhere in `Src/`** — added on the list, unused by the app. Read it as `.Value` if a screen ever needs it |

Loaded in full into `colRawMaterials` on `RequestFormScreen.OnVisible`; `App.OnStart` preloads `FirstN('Raw Materials', 1)` as a schema-shape seed. The picker (`ddMaterial_1`), its `DefaultSelectedItems` and the bulk-code parser all read that collection, never the list directly — pointing `ddMaterial_1.Items` at the list caps the combobox's search at the app's Data row limit, because its search runs in memory (`Search()` is not delegable for SharePoint).

**The app's Data row limit must stay at 2000** (App settings → General) and this list must stay under it — ~700 rows today. A whole-list `ClearCollect` truncates silently, so `OnVisible` follows it with a `Max(colRawMaterials, ID)` vs. delegated-max-`ID` comparison that raises an error notification if it happened. Chunking on `ID` does **not** raise the ceiling — SharePoint delegates only `=` on `ID`, not ranges. See the `colRawMaterials` / `colProjects` note in `CLAUDE.md` for what a real fix past 2000 would require.

**`ddMaterial_1` is a `Classic/ComboBox@2.4.0`, not a `ModernCombobox`** — deliberately, and it is the only raw-material picker in the app. Note the control type carefully: `ComboBox@0.0.51` (what `ddUnit_1` and the receipt dropdowns use) is a *different, newer* control that rejects both properties below with `PA2108: Unknown property`. Only the `Classic/ComboBox@2.4.0` type accepts them — `ddProject_1` on the same screen is the other instance. The requester must be able to find a material by **either** trade name or code, and only the classic control has `SearchFields` (`["Title", "Code"]`), which matches each listed field independently. The modern combobox has no equivalent: it exposes `IsSearchable` and a `SearchText` output but filters against `ItemDisplayText` only, so a code-only match can't surface. Cramming both into one display string doesn't fix it either — that changes what the closed control reads back as the selection, and still depends on undocumented matching behavior. `DisplayFields: =["Title", "Code"]` renders the flyout as trade name over code, so the requester sees *why* a row matched. The classic control's "selections aren't maintained when the gallery scrolls" limitation doesn't apply here: `galLineItems` sizes itself to `CountRows(colLineItems)` and never scrolls internally (`conScrollable` scrolls instead), and the selection is re-derived from `colLineItems` via `DefaultSelectedItems` regardless. Don't "modernize" this control back without solving the two-field search.

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
| `Decision` | Choice | Manager: `Approve`, `Reject` · Executive: `Approve`, `Reject`. **Both roles now write the same two-value vocabulary, so this column needs no new option** — `ManagerReviewScreen` was cut from three options to two and deliberately reuses the Executive's existing `Approve`/`Reject` values rather than introducing `Approved`. The three old Manager values (`Approved (within budget)`, `Needs clarification`, `Exceeds budget / unplanned`) are no longer written by anything; and `Approve with conditions` was likewise dropped from `ExecutiveApprovalScreen`. No app code renders any of the four any more — `RequestDetailScreen`'s decision colour map now only knows `Approve` and `Reject` — so all four can be removed from the Choice column once the old rows are migrated by hand. Note `StepNumber` is what tells a Manager `Approve` from an Executive `Approve` |
| `ApprovalConditions` | Text (multiline) | step 3 — the Executive's **Remark**, required on **both** `Approve` and `Reject` (one box serves both; there is no separate rejection-reason input any more). **The single home for this value** — the duplicate `'RM Procurement Requests'.ConditionsText` was removed, and all three readers (`RequestDetailScreen` via `gExecutiveRemark`, `ProcurementExecutionScreen`, `ExecutivePaymentScreen`) now resolve it off this row. Column name unchanged |
| `RejectionReason` | Text (multiline) | **dead — the string doesn't appear anywhere in `Src/` any more.** Both approval screens have one always-required remark box, so a rejection reason lives in that step's own remark column (`ManagerRemarks` for step 2, `ApprovalConditions` for step 3). Writing it here *as well* would make `RequestDetailScreen`'s approval note line print the same text twice. Old rows are being migrated by hand; the column can be dropped once that's done |
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
| `StepNumber` | Number | `1` Procurement Execution · `2` Accounting Handover (written by `AccountingScreen`'s submit, i.e. at completion — despite the "handover" name it records the accounting step being *done*, not assigned) · `3` Goods Receipt **round 1** · `4` Goods Receipt **round N ≥ 2** · `5` Supplier Follow-up (Procurement, the `Accepted with Adjustment` close-out) · `6` Invoice Submission · **`7` Delivery Batch Created** · **`8` Request Closed** |
| `StepName` | Choice | matches the step. **Two new options must be added on SharePoint**: `Delivery Batch Created`, `Request Closed` |
| `RoundNumber` ⬅ **new** | Number | which receipt round this row belongs to. Step 3 always `1`; step 4 = `N`; step 5 = the round it follows up on. **On a step-7 row it carries the `DeliveryNumber` instead** — the column is reused rather than adding a ninth. **Blank on rows written before the unlimited-rounds change** — every reader coalesces to `If(StepNumber = 3, 1, 2)` |
| `ReceivedBy` ⬅ **new** | Text | steps 3/4 — display name of whoever physically received the goods |
| `ReceiptDate` ⬅ **new** | Date | steps 3/4 |
| `ReceiptStatus` ⬅ **new** | Text | steps 3/4 — copy of the chosen `GoodsReceiptStatus` / `FollowUpReceiptStatus` choice value. Plain Text so both choice sets fit |
| `AcceptanceDecision` ⬅ **new** | Text | steps 3/4 — copy of the chosen acceptance decision. Plain Text, same reason |
| `ExecutedBy` | Lookup→Employee List | |
| `ExecutedAt` | DateTime | |
| `HandoverToID` | Lookup→Employee List | ⚠ **no longer written** — held the Accounting staff picked at step 1 / step 6. Both pickers were removed; safe to delete from the list |
| `HandoverToIDText` | Text | ⚠ **no longer written** — same; safe to delete |
| `SupplierSummary` | Text (multiline) | step 1. Column name unchanged, but the UI now labels it **"SO/PO/INV"** everywhere it's shown (`ProcurementExecutionScreen`'s input — now a single-line `TextInput`, `RequestDetailScreen`'s log-detail dialog, `AccountingScreen`'s reference panel) — it holds a Sales/Purchase Order or Invoice number, not a free-text negotiation summary |
| `PurchaseOrderLink` | Text (URL) | step 1 |
| `Notes` | Text (multiline) | **shared column, meaning depends on `StepNumber`**: step 1 (reject reason) / 2 · steps 3 & 4 = that receipt round's Remarks · step 5 = that round's Procurement follow-up notes |
| `Attachments` | Attachments | receipt photos for the round — step 3 via `frmGRLog_GR`, step 4 via `frmSFU1Log_SFU`; `Patch` alone can't write attachments. One row per round ⇒ photos are naturally kept per round |

**Steps 1, 7 and 8 are written against the parent; steps 2–6 against a delivery.** That split is deliberate: Procurement Execution, "delivery created" and "request closed" are all things that happen *to the request*, so the parent's own history reads as a complete story of the sourcing decision and every delivery spun off it, while each delivery's history holds only its own receiving and invoicing. There can be **many step-7 rows per parent** — one per delivery — so, like step 4, never `LookUp` one expecting uniqueness.

Consequence to know about: a delivery's `colExecutionLog` therefore contains **no step-1 row**, so its detail screen shows no Supplier Summary or PO link — those live on the parent, one click away via the delivery's banner. Its `colApprovalLog`, by contrast, *is* back-filled from the parent (`RequestIDText = Text(gRootRequestID)`), because a delivery heading into Accounting with no visible approval trail at all was an audit regression.

There can be **many step-4 rows per request** (one per receipt round from round 2 on) — never `LookUp` one expecting it to be unique. A step-5 row is written at most once, on the `Accepted with Adjustment` branch. Routing state lives on the request in `ReceiptRoundCount` / `LatestReceiptDecision`, not in the log. The presence of a step-6 row still distinguishes whether `InvoiceSubmissionScreen` has already run for a request.

---

## `'RM Procurement Receipt Rounds'` ⬅ **new list** — one row per (line item × receipt round)

Created for the unlimited-receipt-rounds change: the old fixed `…1` / `…2` columns on `'RM Procurement Line Items'` could only ever hold two rounds. Written by `GoodsReceiptScreen` (round 1) and `SupplierFollowUpScreen` (round N ≥ 2); **only materials with `ReceivedQty > 0` get a row**, so a round where a material wasn't delivered simply has no row for it.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | `R<n> - <material name>` |
| `RequestID` | Lookup→'RM Procurement Requests' | `{Id, Value}` |
| `RequestIDText` | Text | join key — every screen filters on this. Always the **delivery's** ID, since receiving only ever happens on a delivery |
| `LineItemID` | Number | plain number copy of `'RM Procurement Line Items'.ID` (not a Lookup — kept delegable for equality filters and simple to `Sum`/`Filter` locally). The **delivery's** line item |
| `ParentRequestID` ⬅ **new** | Number | copy of the delivery's `ParentRequestID`. Lets the parent gather everything received across all its deliveries in **one** delegable query: `Filter('RM Procurement Receipt Rounds', ParentRequestID = <parent id>)`. Without it you would need `RequestIDText in <list of child ids>`, which does not delegate. Worth indexing |
| `ParentLineItemID` ⬅ **new** | Number | copy of the delivery line item's `ParentLineItemID`. `RequestDetailScreen` sums on this to fill the parent's "Received" column |
| `RoundNumber` | Number | 1 = Goods Receipt, ≥2 = Supplier Follow-up rounds |
| `MaterialName` | Text | denormalized copy so COA/history screens don't have to re-join |
| `Unit` | Text | denormalized copy |
| `ReceivedQty` | Number | quantity received **in this round only** — cumulative received = `Sum` across rounds |
| `BatchNumber` | Text | required on submit |
| `ExpiryDate` | Date | required on submit |
| `QCNumber` | Text | required on submit |
| `RMPKCode` | Text | **derived, never typed** — copied from the raw material's `Code` when `colRoundEntry` is built, shown read-only, written through on submit. Blank if that material has no `Code` in the catalog |
| `COALink` | Text (URL) | required on submit — see "Link to COA completion" in `CLAUDE.md` |
| `COAMissing` | Yes/No | `IsBlank(COALink)` at write time, so now always `false` for newly created rows; delegation-safe filter for the outstanding-COA views, which from here on only match rows created while `COALink` was still optional (see the delegation note under `'RM Procurement Line Items'`) |

`colReceiptRounds` mirrors this list per request and is the only source for "what has been received": `PrevReceivedQty` on the entry screens, `TotalReceivedQty` on `RequestDetailScreen`, the per-round "View receipt details" dialog on all three history screens (`colRoundDetails`), and `ProcurementFollowUpScreen`'s "what was received in Round N" table all read it. This list replaced the deleted `…1`/`…2` line-item columns outright — there is no back-fill or synthetic-row path anywhere.

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
- **`Suppliers`** — zero references anywhere in current code. Removed together with the request-level `PreferredSupplier` field. Not to be confused with **`RM Suppliers`**, the list `'Raw Materials'.Supplier`/`SupplierID` now look up into; that one is live, but only ever reached *through* those Lookup columns — the app never adds it as a data source of its own.

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

Called from `ProcurementExecutionScreen` (2 call sites — Deferred and Via-Requester) and `InvoiceSubmissionScreen` (2 call sites, same two paths). 17 positional args, identical order at all 4 call sites:

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

`ProjectID` is **not** passed from the app. The flow resolves it on its own from the request ID (param 2), so the trigger's 18th parameter — if it still exists — is left unfilled by Power Fx. Don't "fix" a call site by adding `gSelectedRequest.ProjectID` back as an 18th arg.

#### What `Submit_Invoice` actually reads off `'RM Procurement Requests'` — verified 2026-08-13

Checked against the live published flow when the parent/delivery split shipped, because param 2 is now a **delivery's** ID rather than the original request's. **Result: `ProjectID` is the only request column the flow consumes**, so no extra columns had to be copied onto a delivery beyond what it already copies.

- **`Get_ProjectID`** — `GetItem` on `'RM Procurement Requests'` (list GUID `e1bda0b4-8e8e-4fd1-ac5e-752be9df6f3d`), `id = triggerBody()?['number']`. Its output is referenced exactly once, as `item/ProjectID: @body('Get_ProjectID')?['ProjectID']`.
- **`Procurement InvoiceData Attachment`** — `PostItem` into `Procurement_InvoiceData` (GUID `3a7218e7-e10f-406a-9a8a-d4cbbb2ea82c`). Every other field comes straight from `triggerBody()` (i.e. from the app's 17 args), **not** from the fetched request row. Two mappings are worth knowing because the names don't line up: `item/CostCenter` ← `text_10`, which is arg 15, the app's **`InvoiceRegion`**; and `item/Origin` ← `text_12`, arg 17, the source-app literal.
- **`item/RequestID/Id`** and **`item/RequestIDText`** ← `triggerBody()?['number']`, so from now on they hold the **delivery's** ID, not the original request's. The join still resolves (a delivery is a row in the same list) and the delivery's Title names its parent, but anything grouping invoices by original request must hop through `ParentRequestID`. The Actual-Cost panel is unaffected — it groups by `ProjectID`, and now correctly counts N invoices per request instead of 1.
- **`Update RM OfficialInvoiceLink Attachment`** — `Update item` on `'RM Procurement Requests'`, `Id` taken from the Get item, i.e. the **delivery**. So the flow's final `OfficialInvoiceLink` write lands per delivery. **This is the single behaviour that decided same-list over a separate deliveries list**: passing a delivery ID keeps this contract working untouched, while a separate list would have forced either a flow edit or one shared parent link overwritten once per delivery.
- The `Switch` on `text_12` still has the `Raw Materials Procurement App` case that this app's arg 17 matches.

**Returns**: `newinvoicelink` (string) — **never read by the app**, and it doesn't need to be. All 4 call sites do `Set(gSubmitInvoiceResult, Submit_Invoice.Run(...))` and then only `IsError(gSubmitInvoiceResult)`: the variable is purely a "did the flow run" flag, so don't treat it as data or try to display it.

The link handoff works like this instead: the app writes `OfficialInvoiceLink` itself (`wOfficialInvoiceLink` / `locInvoiceURL_ISS`) *before* calling the flow, from the same URL it passes as param 1 — that value is only an interim pointer at the attachment on the request item. **The flow then patches `OfficialInvoiceLink` again** with the final location after it renames/files the invoice. So the app's write is expected to be overwritten, and `newinvoicelink` is deliberately dropped on the Power Fx side. Don't "fix" this by patching the request from the flow's return value.

### `ProcurementNotify.Run(assigneeEmail, assigneeName, requestTitle, requestId, notificationType, deliveryDate, category, sourceApp)`

SharePoint flow name: **`Procurement Notify`** (renamed from `Procurement_Notify_Receipt_Assignee`). **8 positional args, same order at all 8 call sites** — it is the app's general-purpose notifier, not just an assignment notifier. Trigger param names as referenced inside the flow: `text` (1), `text_1` (2), `text_2` (3), `number` (4), `text_3` (5), `text_4` (6), `text_5` (7), `text_6` (8).

| `notificationType` | Call site(s) | Recipient | Channels |
|---|---|---|---|
| `"GoodsReceipt"` | `GoodsReceiptScreen.btnSaveAssignment_GR` | new assignee | email + Teams card |
| `"SupplierFollowUp"` | `SupplierFollowUpScreen.btnSaveAssignment_SFU` | new assignee for the open round (**any round ≥ 2**) | email + Teams card |
| `"Unassigned"` | `btnSaveAssignment_GR`, `btnSaveAssignment_SFU`, `btnIWillReceive_GR`, `btnIWillReceive_SFU` | previous assignee | email only |
| `"ExecutivePayment"` | `ExecutiveApprovalScreen` (over-threshold approval) | `gCurrentEmployee.Email` — the Executive notifies themselves | email + Teams card |
| `"RMProcurementExecution"` | `ExecutivePaymentScreen` (after remittance upload) | hardcoded `"procurement@maxbiocare.com"` — **not** resolved from `'RM User'` | email + Teams card |

- The `GoodsReceipt` and `SupplierFollowUp` branches now render **identical** content (both say "Goods Receipt", no round number), so they can be collapsed into one branch. A Power Automate `Switch` case accepts only one value, so collapse by switching on a normalized Compose rather than by listing two values in one case.
- `category` (arg 7) is **reserved for the request-level Category** and is `""` at every call site, because request-level Category no longer exists (it moved per raw-material). Keep it `""` — do **not** repurpose the slot (e.g. to smuggle a round number through). If the notification ever needs the round, add a **9th** arg and update all 8 call sites in the same edit; Power Fx requires exact arity.
- `sourceApp` (arg 8) is always the literal `"Max Biocare · Raw Materials Procurement"`; the flow prints it in the email header/footer and the card header.
- Connection: `app.admin@maxbiocare.com` pinned in "Run only users" — not invoker-provided.

### `Procurement_Notify_Invoice_Provided.Run(requestTitle, requestId, invoiceUrl)`

3 args. Called once, from `RequesterInvoiceScreen`, after a Requester successfully uploads/re-uploads a corrected invoice — notifies Procurement.

### `Procurement_Notify_Remind_Invoice.Run(requesterEmail, requesterName, requestTitle, requestIdText, rejectedByOrEmpty)`

5 args. Called twice from `InvoiceSubmissionScreen`:
- "Remind Requester" button — last arg `""`.
- "Request Re-upload" button — last arg = `gCurrentEmployee.Title` (the Procurement employee who rejected the submitted invoice).
