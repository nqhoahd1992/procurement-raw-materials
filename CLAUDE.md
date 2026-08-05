# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Max Biocare Raw Materials Procurement** — a Power Apps Canvas App (DesktopOrTablet, 1366×768) that manages the end-to-end procurement request workflow for raw materials. The Power Fx source lives in `.pa.yaml` files.

This folder keeps **only the screen logic + UI** (`Src/`). The generated metadata that the full unpacked `.msapp` would contain (`Controls/`, `References/`, `Resources/`, `Properties.json`, `Header.json`, `AppCheckerResult.sarif`) has been **intentionally removed** — the workflow here is to read/edit Power Fx and paste it into **Power Apps Studio in the browser**, not to repack a `.msapp`. Because of that, this folder can no longer be packed with `pac canvas pack`; re-export from Power Apps if you ever need the full package again.

> Note: the sibling working directory `../ci_cd` is a **separate** Node.js/Docker project, unrelated to this Canvas App. Don't conflate the two.

## Layout

- `Src/*.pa.yaml` — one file per screen (the actual logic + UI). Screen order in `Src/_EditorState.pa.yaml`.
- `Src/App.pa.yaml` — `App.OnStart`: resolves the signed-in user and sets global state.
- `Src/SupplierFollowUpScreen.pa.yaml` — records **one receipt round (round ≥ 2)**, performed by the Requester or the person they assign. Nothing else: the Procurement side of the follow-up moved out to its own screen (below) because the two halves have different actors, and cramming both into one screen made the visibility gating unmanageable once rounds became unbounded.
- `Src/ProcurementFollowUpScreen.pa.yaml` — **Procurement/Admin only**; runs **at most once per request** and only on the `Accepted with Adjustment` branch: remarks + Credit Note upload, which also ends the receiving process. Reached from `RequestDetailScreen`'s "Upload Credit Note →" button. Procurement does **not** sit between ordinary receipt rounds — a `Requires Supplier Follow-up` decision opens the next round immediately.
- `Src/COACompletionScreen.pa.yaml` — per-request screen (Procurement/Admin only, reached via a "Link to COA" button on that request's row in `HomeScreen`) for filling in `COALink` on `gSelectedRequest`'s `'RM Procurement Receipt Rounds'` rows left blank at Goods Receipt / Supplier Follow-up — any round, any number of them. See "Link to COA completion" below — it is intentionally **not** part of the request Status workflow.

## Backend & connectors

All data lives in **SharePoint Online** (`maxbiocare.sharepoint.com/sites/Powerapps`). Lists:
- `'RM Procurement Requests'` — the central request record (status, approvers, costs, invoice, goods-receipt, follow-up fields). No longer carries `Category` or `PreferredSupplier` — both moved to the per-material line-item level (see `'Raw Materials'` below).
- `'RM Procurement Line Items'` — one row per raw material on a request (`MaterialID`/`MaterialName`, `Unit`, `Quantity`). Carries **no receipt data at all** — the old `…1`/`…2` received-quantity/batch/expiry/COA columns were deleted when rounds became unbounded.
- `'RM Procurement Receipt Rounds'` — **one row per (line item × receipt round)**, only for materials actually received in that round. This is what makes an unbounded number of receipt rounds possible; the old fixed `…1`/`…2` columns could hold exactly two.
- `'Raw Materials'` — the raw-material catalog. Only `ID`, `Title` (trade name), `Code`, and `Category` are currently wired into the app; other catalog columns that may exist on SharePoint are not yet referenced anywhere in this code.
- `'RM User'` — maps an employee to a `Role` (Requester / Manager / Executive / Procurement / Accounting / Admin). **This list is now also the app's membership gate** — an employee must have a row here to use the app at all (see "Global state" and the membership note below). `Requester` must exist as an actual selectable value in this list's `Role` Choice column on SharePoint — that's a SharePoint list-settings change (List settings → `Role` column → edit choices), not something a `.pa.yaml` change can do. Because of this gate, the Goods Receipt / Supplier Follow-up "Assign to someone else" pickers (`ddAssignReceiver_GR`/`ddAssignReceiver_SFU`) are filtered to `Employee List` rows whose `ID` is among active `'RM User'` rows' `EmployeeID.Id` — an employee can only be assigned a receiving task if they can actually log into the app to perform it. Don't widen these pickers back to the full `Employee List` without re-adding some other way for the assignee to gain access.
- `'RM Procurement Approval Log'` — manager/executive decisions (StepNumber 2 = Manager, 3 = Executive).
- `'RM Procurement Execution Log'` — procurement/goods-receipt/follow-up/invoice step records (StepNumber 1 = Procurement Execution, 2 = Accounting Handover, 3 = Goods Receipt **round 1**, 4 = Goods Receipt **round N ≥ 2**, 5 = Supplier Follow-up Procurement **for one round**, 6 = Invoice Submission). Steps 3/4/5 carry `RoundNumber` plus the round's header (`ReceivedBy`, `ReceiptDate`, `ReceiptStatus`, `AcceptanceDecision`, `Notes`, `Attachments`). **There are now multiple step-4 and step-5 rows per request** — never `LookUp` one expecting uniqueness.
- `'Employee List'`, `Project_List` (cross-app list from the sibling `project-list` app).
- `Product_Database_SKU_Master` — external product-SKU catalog, used only by `RequestFormScreen`'s optional multi-select `cmbSKU` picker (`SelectMultiple: =true`, writes `'RM Procurement Requests'.RelatedSKU` — a multi-value Lookup column — via `ForAll(cmbSKU.SelectedItems, {Id: ID, Value: Title})`). Not used anywhere else in this app.

- `Procurement_InvoiceData` — cross-app invoice list (owned by the sibling procurement app). **Read-only here**, used solely by the "Project Information" panel on `ManagerReviewScreen` / `ExecutiveApprovalScreen` to compute the related project's Actual Cost: `Filter(Procurement_InvoiceData, ProjectID = gSelectedRequest.ProjectID)` summed by `Currency` over `TotalAmount`. Must be added as a connected data source in Studio. This app never writes to it.

Lists that **no longer exist in the app's code** (removed in a prior refactor — do not reintroduce without checking with the user first): `Suppliers`. If it still exists on SharePoint it is currently orphaned/unused by this app.

Full column-level schema (types, choice values, join keys): `docs/sharepoint-schema.md`.

**Power Automate** flows called from Power Fx:

Invoice flows (called from `ProcurementExecutionScreen` and `InvoiceSubmissionScreen`):
- `Parse_Invoice.Run(invoiceUrl, requestId)` — AI invoice extraction.
- `Submit_Invoice.Run(...)` — 18 positional args; writes the official invoice. Param 14 is the invoice **Description** field value, param 17 is the source app name (`"Raw Materials Procurement App"`), param 18 is `gSelectedRequest.ProjectID` (not reserved/empty slots — see `docs/sharepoint-schema.md` for the full param table).

General-purpose notification flow — SharePoint flow name **`Procurement Notify`** (renamed from `Procurement_Notify_Receipt_Assignee`), referenced in Power Fx as `ProcurementNotify`:
- `ProcurementNotify.Run(assigneeEmail, assigneeName, requestTitle, requestId, notificationType, deliveryDate, category, sourceApp)` — **8 args**, identical order at all **8 call sites**. The flow branches on `notificationType`:
  - `"GoodsReceipt"` — new assignee must perform Goods Receipt & Acceptance (round 1). `GoodsReceiptScreen.btnSaveAssignment_GR`.
  - `"SupplierFollowUp"` — new assignee must perform the open receipt round (**round ≥ 2, any round** — the email is deliberately round-agnostic, since the app passes no round number). `SupplierFollowUpScreen.btnSaveAssignment_SFU`.
  - `"Unassigned"` — previous assignee no longer needs to act (email only, no Adaptive Card). Four call sites: `btnSaveAssignment_GR` / `btnSaveAssignment_SFU` (before assigning someone else) and `btnIWillReceive_GR` / `btnIWillReceive_SFU` (requester takes the task back).
  - `"ExecutivePayment"` — `ExecutiveApprovalScreen`, when an approved request is over-threshold: the Executive notifies **themselves** (`gCurrentEmployee.Email`) to go process the payment.
  - `"RMProcurementExecution"` — `ExecutivePaymentScreen`, after the remittance upload, notifies Procurement. **Recipient is hardcoded `"procurement@maxbiocare.com"`**, not resolved from `'RM User'` — verify that address is a real distribution list, or this notification goes nowhere.
  - `category` (arg 7) is **reserved for the request-level Category** and is currently `""` everywhere, because request-level Category no longer exists (it moved per raw-material). Keep it `""` — do **not** repurpose the slot to carry anything else (e.g. a round number). If the email ever needs the round, add a **9th** arg and update all 8 call sites together; Power Fx requires exact arity, so it can't be added incrementally.
  - `sourceApp` (arg 8) is always the literal `"Max Biocare · Raw Materials Procurement"`; the flow prints it in the header and footer.
  - Connection: `app.admin@maxbiocare.com` pinned in "Run only users" — not invoker-provided.

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
- `colLineItemsDetail` — per-request line items loaded from `'RM Procurement Line Items'` on GoodsReceiptScreen / SupplierFollowUpScreen / RequestDetailScreen. Now a **read-only staging collection**: no screen patches receipt data back onto it. Each screen projects it into a purpose-built collection instead (below).
- `colReceiptRounds` — the request's `'RM Procurement Receipt Rounds'` rows (one per line item × round). Single source of truth for everything received: `PrevReceivedQty` on the entry screens, `TotalReceivedQty` on `RequestDetailScreen`, and the per-round receipt-details dialog (`colRoundDetails`, below).
- `colRoundEntry` — the blank per-round data-entry buffer on GoodsReceiptScreen / SupplierFollowUpScreen: `{LineItemID, MaterialID, MaterialName, Unit, OrderedQty, PrevReceivedQty, ReceivedQty, BatchNumber, ExpiryDate, QCNumber, RMPKCode, COALink}`. `MaterialID` is a **plain number** here (`li.MaterialID.Id`), so the catalog lookup is `LookUp('Raw Materials', ID = ThisItem.MaterialID)` with no `.Id`. Built with `ForAll(colLineItemsDetail As li, {...})` — the `As li` alias is required, since `ThisRecord` inside the nested `Filter(colReceiptRounds, LineItemID = li.ID)` would otherwise rebind to the inner scope.
- `colRoundHistory` — the per-round header timeline, built from `'RM Procurement Execution Log'` Step 3/4/5 rows sorted by `ID`: `{ID, RoundNumber, StepNumber, StepLabel, ActorName, ActionDate, ReceiptStatus, AcceptanceDecision, Notes}`. `ActorName`/`ActionDate` `Coalesce` the receipt fields against `ExecutedBy`/`ExecutedAt` because Step-5 rows have no receiver or receipt date. Backs the history galleries on SupplierFollowUpScreen / ProcurementFollowUpScreen / RequestDetailScreen.
  - **Do not try to project `Attachments` into this collection.** `Attachments` is not part of the row scope of a `ForAll` over a SharePoint list — `lg.Attachments` evaluates to an Error there, so `Concat(lg.Attachments, DisplayName, …)` fails with *"'DisplayName' isn't recognized"* and adding an `As` alias just moves the error to *"'att' isn't recognized"*. An attachment column is only reachable on a **full record fetched by `LookUp`**, which is what the dialog below does.
- `colReceiptPhotos` / `gShowReceiptPhotos` / `gPhotoRoundLabel` — the **receipt-attachments dialog**. A round can have any number of attachments and a gallery-inside-a-gallery with a fixed `TemplateSize` would clip them, so each history row carries a "View attachments" button that does `ClearCollect(colReceiptPhotos, LookUp('RM Procurement Execution Log', ID = ThisItem.ID).Attachments)` and flips `gShowReceiptPhotos`. The button is gated on `ThisItem.StepNumber <> 5`, **not** on an attachment count — because of the row-scope limitation above the row can't know how many files a round has. That gate is exact rather than approximate: Step 3 and Step 4 rows always carry photos (both submits reject an empty `attGRPhotos`/`attSFU1Photos`), and Step 5 rows never carry any, because the Credit Note is attached to the **request** by `formCreditNote_PFU`, not to the log row. The Credit Note stays reachable from `RequestDetailScreen`'s request-level `rowCreditNote` link. The dialog still shows "No files were attached to this round" as a fallback. All three history screens carry an identical scrim + dialog pair (`rectPhotoOverlay_*` / `conPhotoDialog_*`), immediately followed by the receipt-details pair below — the **last four screen-level children** — z-order in Canvas is child order, so they must stay last. The dialog lists file names; each one `Launch(ThisItem.AbsoluteUri, {}, LaunchTarget.New)`. `gShowReceiptPhotos` is reset to `false` in each screen's `OnVisible`. `App.OnStart` seeds `colReceiptPhotos` from `First('RM Procurement Execution Log').Attachments` purely to fix its schema at design time (same trick as `FirstN('Raw Materials', 1)`).
- `colRoundDetails` / `gShowRoundDetails` / `gDetailRoundLabel` — the **receipt-details dialog**, the same lightbox pattern as the attachments one above and living right next to it on all three history screens (`rectRoundDetailOverlay_*` / `conRoundDetailDialog_*`, the last two screen-level children). Each history row's "View receipt details" button projects that round's `colReceiptRounds` rows into `colRoundDetails` (`{LineItemID, MaterialName, Unit, OrderedQty, ReceivedQty, BatchNumber, ExpiryDate, QCNumber, RMPKCode, COALink}`) and flips `gShowRoundDetails`. `OrderedQty` comes from `LookUp(colLineItemsDetail, ID = rr.LineItemID).Quantity`, which is why `ProcurementFollowUpScreen.OnVisible` now loads `colLineItemsDetail` too. Gated on `ThisItem.StepNumber <> 5` (it shares the row's `rowHistAttachments*` container): Step-5 rows are Procurement's Credit Note close-out and record no receipt lines of their own — the round's lines belong to that round's Step-3/4 row. The dialog is 1200 wide with its own column-header row and shows "No receipt lines were recorded for this round." when empty; `COALink` renders as an "Open COA" link or a grey "Not linked". `gShowRoundDetails` is reset to `false` in each screen's `OnVisible`, and `App.OnStart` seeds `colRoundDetails` with a literal record for its design-time schema. The dialog's inner controls are named `*Dlg*` (`rowDlgHeader_*`, `lblDlgColUnit_*`, `galDlgRoundDetails_*`, …) rather than `*RoundDetail*`/`lblRD*` because `ProcurementFollowUpScreen` already uses those names for its always-visible "What was received in Round N" table — Power Apps rejects the paste with `PA2110: An entity with name … already exists`. Keep the `*Dlg*` prefix on all three screens so the blocks stay copy-identical. It is read-only — filling a missing COA link still happens only on `COACompletionScreen`.
- `colLineItemsSummary` — RequestDetailScreen's line-items gallery source: `{ID, MaterialID, MaterialName, Unit, Quantity, TotalReceivedQty}` where `TotalReceivedQty` sums `colReceiptRounds` across all rounds. The gallery is a **flat, full-width, non-selectable table** (Trade Name + a `Code · Category · Supplier` subtitle, Unit, Req. Qty, Received) — there is no row selection and no `gSelectedLineItem` any more. The old side panel (`colLineItemDetailPanel` / `galLineItemRounds`, ~400 lines, driven by an eye-icon `btnViewDetails` per row) showed **the same receipt data pivoted by material** as the receipt-details dialog below does by round, so it was removed rather than kept in parallel: with ~5–10 materials but only 1–2 rounds per request, the round-first view answers the same questions in one click instead of one click per material, and it sits next to that round's "View attachments" button. Don't reintroduce a per-material drill-down without a concrete reason the round-first table can't cover — if one appears, the honest fix is probably a material column *inside* the round dialog, not a second panel.
- `colCOAOutstanding` — `gSelectedRequest`-scoped `'RM Procurement Receipt Rounds'` rows where `COAMissing` is true, backing `COACompletionScreen`; seeded with a literal placeholder record in `App.OnStart` (schema placeholder, same reason as `colLineItemsDetail`'s) and fully reloaded on `COACompletionScreen.OnVisible` and again after each save. Built via `ForAll(Filter(...), {...})` record literals rather than `AddColumns` — the Studio "Preview" YAML/code-paste editor failed to parse `AddColumns` here (rejected the new-column-name string arguments), so every column is instead just named directly in the `ForAll` record. `COARound` is the row's `RoundNumber`; `ReceivedQty`/`BatchNumber`/`ExpiryDate`/`QCNumber`/`RMPKCode` are read-only copies shown for reference next to the input; `NewCOALink` is the working input value before save. The receipt-round row `ID` is genuinely unique here (one row per line item **per round**), so the input's `UpdateIf` keys off `ID` — the `RowKey`/`Source` workaround the old two-column model needed is gone.
- `gProjectInfo` / `colProjectInvoices` / `gShowRateInfo` — set in `ManagerReviewScreen.OnVisible` and `ExecutiveApprovalScreen.OnVisible` (**not** in `App.OnStart`), backing the "Project Information" panel: `gProjectInfo` is the `LookUp(Project_List, ProjectID = gSelectedRequest.ProjectID)` row (blank when the request isn't tied to a project), `colProjectInvoices` the matching `Procurement_InvoiceData` rows, `gShowRateInfo` the "Currency Exchange Reference" dialog toggle (reset to `false` on every screen entry). The AUD conversion rates behind the panel's Grand Total are hardcoded in a `Switch(Upper(Currency), ...)` inside the `lblProjectGrandTotal_MR`/`_EA` labels and mirrored as plain text in the dialog — change both together, and keep them in sync with the same table in the sibling `projects` app's `ViewProjectScreen`.
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
   │ Approve / Approve with conditions (ConditionsText), then:
   │     Currency <> "AUD" OR EstimatedCost > 10000 → stays "Pending Executive" + isExecutivePayment = true
   │                                                   (shown as "Pending Payment From Executive" in UI only —
   │                                                   real Status string is unchanged) → ExecutivePaymentScreen
   │     otherwise                                  → Pending Procurement
   ▼
Pending Procurement ──(ProcurementExecutionScreen)──┐
   │ Reject                      → Rejected
   │ Proceed                     → Goods Receipt & Acceptance
   │   (sets InvoiceMode: "Direct" | "Deferred" | "ViaRequester";
   │    InvoiceSubmitted = true only if the invoice was processed inline)
   ▼
Goods Receipt & Acceptance ──(GoodsReceiptScreen — receipt ROUND 1, ExecutionLog Step 3)──┐
   │ Always ends this screen. Writes ReceiptRoundCount = 1, LatestReceiptDecision = <decision>.
   │ Only 2 options — there is no "Rejected" at goods receipt; a bad delivery enters the loop.
   │ Accepted                    → Fulfillment = "Fulfilled"
   │                             → Pending Invoice (if !InvoiceSubmitted) else Pending Accounting
   │ Requires Supplier Follow-up → Pending Supplier Follow-up, receipt round 2 opens immediately
   ▼
Pending Supplier Follow-up ── THE RECEIVING LOOP, unbounded ──┐
   │
   │  ┌─ SupplierFollowUpScreen — Requester or SFU1AssignedToID, ExecutionLog Step 4, RoundNumber = N
   │  │     shown when LatestReceiptDecision = "Requires Supplier Follow-up"; the round being
   │  │     entered is ReceiptRoundCount + 1. Receiver is re-chosen every round.
   │  │     Writes ReceiptRoundCount = N, LatestReceiptDecision = <decision>, clears SFU1AssignedToID.
   │  │
   │  │     Requires Supplier Follow-up → stays Pending Supplier Follow-up
   │  │                                   → round N+1 opens immediately, NO Procurement step ─┐
   │  │     Accepted                    → Fulfillment = "Fulfilled"                           │
   │  │                                   → Pending Invoice / Pending Accounting  (loop ends) │
   │  │     Accepted with Adjustment    → stays Pending Supplier Follow-up                    │
   │  │                                   → hands over to Procurement ↓                       │
   │  │                                                                                       │
   │  └───────────────────────────────────────────────────────────────────────────────────────┘
   │
   │  ┌─ ProcurementFollowUpScreen — Procurement/Admin, ExecutionLog Step 5, RoundNumber = N
   │  │     Runs at most ONCE per request, and only on the "Accepted with Adjustment" branch.
   │  │     shown when Status = "Pending Supplier Follow-up" && LatestReceiptDecision = "Accepted with Adjustment"
   │  │     Requires Remarks + a Credit Note attachment. Writes CreditNote,
   │  │     Fulfillment = "Fulfilled with Adjustment", and ends the receiving process:
   │  │     → Pending Invoice (if !InvoiceSubmitted) else Pending Accounting
   │  └─
   ▼
Pending Invoice ──(InvoiceSubmissionScreen [Procurement/Admin] or RequesterInvoiceScreen [Requester])──
   │ InvoiceSubmissionScreen submit (ExecutionLog StepNumber 6) → Pending Accounting
   │   (it never bounces back to Pending Supplier Follow-up: under the loop above, "Pending Invoice"
   │    is only ever reached *after* the final Procurement follow-up, so the old `locSFUStep2Pending`
   │    variable was removed outright.)
   │ RequesterInvoiceScreen only uploads RequesterInvoiceURL + notifies Procurement — never changes Status
   ▼
Pending Accounting ──(AccountingScreen)──
   │ Submit → Completed   (this is the ONLY place Status is ever set to "Completed")
   ▼
Completed
```

**Executive-payment sub-flow** (`isExecutivePayment` Yes/No field on `'RM Procurement Requests'`): when Executive approves a request that is over-threshold (`Currency <> "AUD" || EstimatedCost > 10000` — `USD` always qualifies since this app only offers `AUD`/`USD`, no FX conversion for the AUD case, just the plain 10,000 cutoff), the request does **not** advance to "Pending Procurement" — it stays `Status = "Pending Executive"` with `isExecutivePayment = true`, and `RequestDetailScreen`/`HomeScreen` display it as **"Pending Payment From Executive"** purely as a computed label (same pattern as the "Supplier Follow-up (Step 1/2)" sub-status — the real `Status` value never changes). The Executive then uses `ExecutivePaymentScreen` (reached via `RequestDetailScreen`'s "Process Payment →" button, shown only when `isExecutivePayment = true`) to upload a remittance advice document; submitting patches `RemittanceURL` and moves `Status` to `"Pending Procurement"` (keeping `isExecutivePayment = true` for history). Unlike the sibling `procurement-procedure` app, there is no Manager fast-track to worry about here — every request always passes through `ExecutiveApprovalScreen` regardless of decision (see `ManagerReviewScreen`'s hardcoded `Status: {Value: "Pending Executive"}`), so the threshold only needs to be checked in one place.

`RemittanceURL` is shared between two independent producers: `ExecutivePaymentScreen` (this sub-flow) and `ProcurementExecutionScreen`'s own "Remittance Advice Document" upload (Path C / `locIsViaRequester`, when Procurement proceeds with a requester-supplied invoice). When `isExecutivePayment = true`, `ProcurementExecutionScreen` hides its own remittance upload requirement entirely (Executive's upload already satisfies it) and reuses the existing `RemittanceURL` instead of asking Procurement to attach a second document — see `rowFormRemittance_PE` / `rowExecutiveRemittanceInfo_PE` and the `wURL` branch in `formRemittance.OnSuccess`.

Routing relies on these status strings being exact and consistent across `HomeScreen` (filters + gallery `Items` per-role filter), each action screen, and the `Switch`/`If` color maps. **When changing status names or the flow, update every screen that references the string** — there is no shared constant.

### The receiving loop's state — two fields on the request, not the log

`Status` alone can't say where inside "Pending Supplier Follow-up" a request is, and a per-row `LookUp` into the execution log is non-delegable and too slow for `HomeScreen`'s gallery. The loop's position therefore lives in two plain request columns:

| Field | Meaning |
|---|---|
| `ReceiptRoundCount` | receipt rounds submitted so far — the round being entered next is `+1` |
| `LatestReceiptDecision` | the most recent round's acceptance decision (plain **Text**, so both the round-1 and round-N option sets fit) |

Inside `Status = "Pending Supplier Follow-up"` there are exactly two states:

| `LatestReceiptDecision` | Meaning | Screen |
|---|---|---|
| `Requires Supplier Follow-up` | receipt round `ReceiptRoundCount + 1` is open | `SupplierFollowUpScreen` (Requester / `SFU1AssignedToID`) |
| `Accepted with Adjustment` | Credit Note pending | `ProcurementFollowUpScreen` (Procurement/Admin) |

**Always gate on the pair `(Status, LatestReceiptDecision)`, never on the decision alone** — `gSFURoundOpen` and `gPFUPending` both do. The reason is that `LatestReceiptDecision` is a *historical* field: it records what the latest round decided and is never cleared. After Procurement uploads the Credit Note it still reads `Accepted with Adjustment` forever, so a gate reading only the decision would re-open that screen indefinitely. What actually consumes the pending state is `Status` moving to `Pending Invoice` / `Pending Accounting`; the decision only says *which kind* of work was pending.

The receipt-round side is self-clearing for a different reason — the decision is **overwritten** on every submit, so `Requires Supplier Follow-up` always describes the newest round and therefore means "the next round hasn't been recorded yet". Round N+1's submit replaces it. The `Status` half of that gate is defensive rather than load-bearing, but both gates are written the same way so the invariant is enforced rather than assumed.

`Accepted` never leaves a request in this status at all — whichever screen records it routes straight to Pending Invoice / Pending Accounting. That is also why **no third counter is needed**: an earlier design had Procurement follow up after *every* round and needed a `FollowUpRoundCount` to track whose turn it was. That column was dropped once Procurement was scoped down to the single `Accepted with Adjustment` branch.

`ReceiptRoundCount` is read as `Coalesce(ReceiptRoundCount, 0)` on `HomeScreen`, `RequestDetailScreen`, `SupplierFollowUpScreen` and `ProcurementFollowUpScreen` — purely so a request that hasn't reached Goods Receipt yet reads as `0` rather than blank, **not** as a migration fallback. There is no legacy read path anywhere: the columns those fallbacks used were deleted. Keep the four copies in sync.

Both submit buttons re-read the request and abort if it moved under them — `btnSubmitStep1_SFU` checks `ReceiptRoundCount`, `btnSubmit_PFU` checks that `Status`/`LatestReceiptDecision` still say a Credit Note is owed — so two people acting at once can't create a duplicate round or a duplicate close-out.

`SkippedManagerReview` on `'RM Procurement Requests'` and the `ExecutiveApprovalScreen` "Manager Review Skipped" banner (`rowManagerSkipped`, gated on `gSelectedRequest.SkippedManagerReview`) are now **write-only false going forward** — `RequestFormScreen` always writes `false` since every request goes through both approval levels. The field/banner are kept only so older requests submitted before this change (where it may be `true`) still display correctly; don't remove them.

`CostCenter` and `DeliveryLocation` on `RequestFormScreen` are no longer user-selectable — both are hardcoded to `"Port Melbourne Warehouse"` (read-only labels `lblCostCenterValue_1`/`lblDeliveryLocationValue_1`), and `InvoiceRegion` is hardcoded to `"AU"` accordingly. `Currency` is a manual `ddCurrency_1` dropdown (`AUD`/`USD`, default `AUD`) — no longer derived from Cost Center.

### Link to COA completion (decoupled from the Status workflow)

`COALink` on `'RM Procurement Receipt Rounds'` is **optional** at submit time on `GoodsReceiptScreen`/`SupplierFollowUpScreen` — neither screen's submit validation blocks on it, so a request keeps advancing through its normal `Status` (and the receiving loop keeps turning) even if it's left blank. Completing it later does **not** re-route the request or touch `Status` at all; it's handled entirely outside the workflow via `Src/COACompletionScreen.pa.yaml`, a per-request screen (Procurement/Admin only) reached via a "Link to COA" button on that request's row in `HomeScreen` (`btnCOACompletion_Request`, in the row's `rowActions` bar below the description — `Visible` only when the signed-in user is Procurement/Admin, `ThisItem.Status.Value <> "Rejected"`, and that request has at least one outstanding row). The screen re-resolves `gSelectedRequest`, loads only that request's outstanding rows, lets Procurement fill in the missing link per row, and patches it back directly (clearing the matching missing-flag) — no execution-log row, no Status change. A companion standalone Power Automate flow (not called from Power Fx — `RM Procurement - Update COA Reminder for Procurement`, documented in `docs/daily-coa-completion-reminder-flow.md`) sends a daily digest email to every active `'RM User'` row with `Role = "Procurement"`, listing the **requests** that still have at least one `'RM Procurement Receipt Rounds'` row with `COAMissing = Yes` (that flow queries SharePoint directly and is unaffected by how the Canvas screen is scoped). It has been repointed at the new list and no longer references the deleted `COA1Missing`/`COA2Missing` columns. Note it deliberately does **not** exclude Rejected requests — that set is empty in practice, since a request can only be Rejected before any receipt round exists.

`colCOAOutstanding` has exactly one source — `'RM Procurement Receipt Rounds'` rows with `COAMissing` — so the receipt-round row `ID` is its unique key and the input's `UpdateIf` keys off it directly. The old `RowKey`/`Source` disambiguation is gone: back when round data lived in `…1`/`…2` columns a single line-item `ID` could produce two outstanding rows, but each round is now its own row in the new list.

**Why the `COAMissing` flag exists (delegation):** filtering on `IsBlank(COALink)` triggers a Power Apps delegation warning that no formula rewrite can clear — the SharePoint connector never delegates `IsBlank()` on a Text column. `COAMissing` is a Yes/No column computed and written locally (fully delegation-safe to filter on) at the same points the link itself is written: `GoodsReceiptScreen.btnSubmit_GR` and `SupplierFollowUpScreen.btnSubmitStep1_SFU` set it on the receipt-round rows they create; `COACompletionScreen`'s save clears it. See `docs/sharepoint-schema.md` for the full column notes.

`btnCOACompletion_Request.Visible` on `HomeScreen` uses `Not(IsEmpty(Filter(...)))` against both sources, not `CountRows(Filter(...)) > 0` — `CountRows` over a `Filter` against a SharePoint data source is itself non-delegable (separate from whatever's inside the `Filter`), while `IsEmpty` is. Follow this `Not(IsEmpty(...))` pattern for any future "does at least one matching row exist" check against a SharePoint list; `CountRows(Filter(...))` is fine only when the source being filtered is a local collection (e.g. `colReceiptRounds`, `colRoundEntry`, `colCOAOutstanding`), never a live data source.

### Accounting is a shared queue, not an assignment

There is deliberately **no way to assign a request to a specific Accounting person.** `AccountingHandlerID` is written in exactly one place — `AccountingScreen`'s submit, as `gCurrentEmployee` — so it means *"who completed the accounting step"* and stays blank until the request is `Completed`. `RequestDetailScreen` shows it as "Accounting Completed By".

`ProcurementExecutionScreen` and `InvoiceSubmissionScreen` used to carry an "Assign to Accounting Staff *" picker (`ddAccountingHandler_PE` / `ddAccountingHandler_ISS`, both required to submit) that wrote this field plus the log's `HandoverToID`. All of it was removed, because the assignment was never real:
- It was overwritten twice downstream — by `InvoiceSubmissionScreen`, then unconditionally by `AccountingScreen` with whoever actually clicked submit.
- It gated nothing. `HomeScreen`'s Accounting branch filters on **Status**, not on the assignee, so every Accounting user sees every request in `Pending Accounting` / `Goods Receipt & Acceptance` / `Pending Supplier Follow-up` / `Completed`; `AccountingScreen` has no assignee check either.
- It was picked far too early to be meaningful — at Procurement time the request is only entering Goods Receipt, and with unbounded receipt rounds `Pending Accounting` can be arbitrarily far away.

Don't reintroduce a picker without also making it load-bearing: gate `AccountingScreen`, filter `HomeScreen` by assignee, and stop `AccountingScreen` from overwriting the field. Related gap, currently accepted: **nothing notifies Accounting at any point**, not even when `Status` becomes `Pending Accounting` — they discover work by looking at `HomeScreen`.

## Role-based visibility (HomeScreen)

The gallery `Items` filters `'RM Procurement Requests'` differently per `gUserRole` (always further filtered by `IsBlank(gStatusFilter) || Status.Value = gStatusFilter`):
- **Manager** → requests where `ManagerApproverID.Id = gCurrentEmployee.ID`, **or** their own requests (`RequesterEmail = User().Email`).
- **Procurement** → requests in status `"Pending Procurement"`, `"Pending Invoice"`, `"Pending Accounting"`, `"Goods Receipt & Acceptance"`, `"Pending Supplier Follow-up"`, `"Completed"`, `"Rejected"`, or their own requests. Does not include `"Pending Manager"`/`"Pending Executive"`.
- **Accounting** → a narrower version of Procurement's list: `"Pending Accounting"`, `"Goods Receipt & Acceptance"`, `"Pending Supplier Follow-up"`, `"Completed"`, or their own requests (no `"Pending Procurement"`, `"Pending Invoice"`, or `"Rejected"`).
- **Executive / Admin** → all requests, unfiltered.
- **Requester (default, and any unrecognized role)** → own requests (`RequesterEmail = User().Email`), plus requests where they're the current Goods Receipt or receipt-round assignee (`GRAssignedToID.Id = gCurrentEmployee.ID || SFU1AssignedToID.Id = gCurrentEmployee.ID`). Note `SFU1AssignedToID` is cleared at the end of every round, so this only matches while a round is actually open and assigned to them.

The status pill for `"Pending Supplier Follow-up"` is a **computed sub-status**, same pattern as "Pending Payment From Executive" — the real `Status` string never changes inside the loop. It renders `"Supplier Follow-up (Credit Note)"` when `LatestReceiptDecision = "Accepted with Adjustment"`, otherwise `"Goods Receipt (Round <n+1>)"`. It reads only `ThisItem.*` fields — no `LookUp` — which is both delegable and faster than the old Step-4 log probe it replaced.

Filter buttons and "+ New Request" are **not** role-gated — every user who passes the membership gate (`Not(IsBlank(gCurrentEmployee.ID)) && Not(IsBlank(gCurrentUser.ID))`) sees the same filter bar and "+ New Request" button; only the underlying gallery `Items` differ per role. Keep the per-role `Items` filter in sync when adding new statuses, but don't assume the buttons themselves need per-role `Visible` logic — they currently don't have any.

## Pending manual SharePoint work for the unlimited-rounds change

None of this can be done from `.pa.yaml`. **The app will not run correctly until all of it is applied**, and the new screens must be added as data sources in Studio.

1. **Create list `'RM Procurement Receipt Rounds'`** with the columns in `docs/sharepoint-schema.md`, and add it as a data source in Studio.
2. **`'RM Procurement Requests'`** → add `ReceiptRoundCount` (Number) and `LatestReceiptDecision` (Single line of text). Two columns, not three — there is no `FollowUpRoundCount`.
3. **`'RM Procurement Requests'.FollowUpAcceptanceDecision`** → add `Requires Supplier Follow-up` as a third Choice option. Without it the loop can never continue past round 2.
4. **`'RM Procurement Execution Log'`** → add `RoundNumber` (Number), `ReceivedBy` (text), `ReceiptDate` (Date), `ReceiptStatus` (text), `AcceptanceDecision` (text).
5. **Delete the superseded columns.** There was no live receipt data when this change was made, so they were removed outright rather than kept as fallbacks — **the app has no legacy read path**, and conversely it will not compile if any deleted column is still referenced.
   - `'RM Procurement Line Items'` → delete all 14: `ReceivedQty1` `BatchNumber1` `ExpiryDate1` `QCNumber1` `RMPKCode1` `COALink1` `COA1Missing` `ReceivedQty2` `BatchNumber2` `ExpiryDate2` `QCNumber2` `RMPKCode2` `COALink2` `COA2Missing`.
   - `'RM Procurement Requests'` → delete 10: `GoodsReceiptBy` `GoodsReceiptDate` `GoodsReceiptRemarks` `GoodsReceiptAt` `FollowUpReceiptBy` `FollowUpReceiptDate` `FollowUpRemarks` `FollowUpReceiptAt` `SupplierFollowUpNotes` `FollowUpCompletedAt`. (`FollowUpCompletedAt` was write-only even before this change — the same timestamp is `ExecutedAt` on the final Step-5 log row.)
   - `'RM Procurement Requests'` → also delete all four receipt Choice columns: `GoodsReceiptStatus`, `GoodsAcceptanceDecision`, `FollowUpReceiptStatus`, `FollowUpAcceptanceDecision`. Their four dropdowns now hold the options inline as literal tables, so nothing references the columns.
   - `'RM Procurement Execution Log'` → delete `HandoverToID` and `HandoverToIDText`. They held the Accounting staff picked ahead of time on `ProcurementExecutionScreen` / `InvoiceSubmissionScreen`; both pickers were removed (see "Accounting is a shared queue" below), so nothing writes or reads them.
6. ~~**`docs/daily-coa-completion-reminder-flow.md`'s flow** queries `'RM Procurement Line Items'.COA1Missing`/`COA2Missing`, which no longer exist.~~ **Done** — `RM Procurement - Update COA Reminder for Procurement` now reads `'RM Procurement Receipt Rounds'` with `COAMissing eq 1`, digests at request level, and guards against sending an empty email. See `docs/daily-coa-completion-reminder-flow.md`.

## Conventions

- All UI text, field names, comments, and code are **English** (per global instruction). Vietnamese appears only as SharePoint system-column internal names and the `vi-VN` UserLocale.
- Patches use the `With({wPatched: Patch(...)}, If(IsBlank(wPatched.ID), Notify(error), ...success...))` pattern to guard against write failures — follow it for new writes.
- **Attachment cards bound to `'RM Procurement Requests'.{Attachments}` always set `Default: =Blank()`.** That column belongs to the request, not to a step, and six screens append to it (request form, `RequesterInvoiceScreen`, `ProcurementExecutionScreen` ×3, `InvoiceSubmissionScreen`, `ProcurementFollowUpScreen`). A blank Default means the control starts empty and only reports the file just added — which is what makes the standard `If(CountRows(attX.Attachments) = 0, Notify("… is required", …), Set(gPending…Name, Last(attX.Attachments).Name); SubmitForm(…))` guard correct, and what lets `OnSuccess` build the file URL from `Last(...).Name`. Existing attachments are untouched by the submit. Do **not** point Default at the record's attachments (`gSelectedRequest.Attachments` / `ThisItem.Attachments`) on these cards: the control then edits that same collection in place, so any "did the count grow?" check compares a value against itself and rejects every submit — this is exactly how `btnSubmit_PFU` came to reject Credit Notes that had in fact been attached. (The Goods Receipt / Supplier Follow-up photo cards are a different case — they bind to the *log row's* attachments via a gallery `ThisItem`, and use `gGRPendingAttachments`/`gSFU1PendingAttachments` to survive a re-render.)
- `RequestIDText` (text copy of the request ID) is used to look up log rows. **Steps 1/2/6 are unique per request and can still be `LookUp`ed; steps 3/4/5 are not** — there is one row per receipt round, so filter and sort them instead.
- Colors are inline `RGBA(...)`; brand purple is `RGBA(83, 74, 183, 1)`.
- The four receipt dropdowns hold their options as **inline literal tables**, not `Choices()` — the backing SharePoint columns are gone. A literal `["a", "b"]` is a one-column table whose column is named `Value`, so `.Selected.Value` reads the same as it did with `Choices()`. Changing an option is now a code edit pasted into Studio; and since `LatestReceiptDecision` / the log's `AcceptanceDecision`/`ReceiptStatus` store these as plain text, renaming an option also means updating every `= "..."` comparison in the receiving loop.
  - `ddReceiptStatus_GR`, `ddReceiptStatus2_SFU` → `Fully Received` · `Partially Received` · `Incorrect Items` · `Damaged Items`
  - `ddAcceptanceDecision_GR` (round 1) → `Accepted` · `Requires Supplier Follow-up`
  - `ddAcceptanceDecision2_SFU` (round ≥ 2) → `Requires Supplier Follow-up` · `Accepted` · `Accepted with Adjustment`
  - Round 1 deliberately has **no `Rejected`** — a bad delivery goes into the receiving loop instead of terminating the request. The `Rejected` *Status* value is untouched; Manager/Executive/Procurement still set it.
