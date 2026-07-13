# Power Automate — Daily Goods Receipt Reminder Flow

Design/build guide for a **scheduled cloud flow** that reminds the responsible person to receive goods and enter goods details for any request stuck in the Goods Receipt / Supplier Follow-up stage.

> This is a **standalone** Power Automate flow built in the portal. It is **not** called from the Power App and requires **no** `.pa.yaml` change. This document is the build/reference guide only.

## Why

When a `'RM Procurement Requests'` row moves to **`Goods Receipt & Acceptance`** or **`Pending Supplier Follow-up`**, the assigned person must open the Power App and enter the goods details on each line item (Received Qty, Batch Number, QC Number, Expiry Date, RM/PK Code, Link to COA). The app only sends a **one-time** email when the assignee is set/changed (`Procurement_Notify_Receipt_Assignee`). If the person forgets, nothing follows up, so requests can sit in these two statuses for days. This flow adds a daily nudge.

## Behavior (agreed requirements)

- **When to remind:** every run, for **any** request currently in `Goods Receipt & Acceptance` or `Pending Supplier Follow-up`. No overdue threshold — being in the status already means receiving is incomplete (the app advances `Status` automatically once the receiving form is submitted), so there is no need to inspect individual line items.
- **Email content:** **short** — request title, status, required delivery date, and a link to open the request. Does **not** list line items ⇒ no need to query `'RM Procurement Line Items'`.
- **Recipient:** only the person who must receive the goods (GR/SFU assignee; if unassigned, the requester). No CC.
- **Schedule:** 08:00 Melbourne time, Monday–Friday.

## Recipient resolution (core logic)

On `'RM Procurement Requests'` (see [sharepoint-schema.md](sharepoint-schema.md)), the assignee columns are **Lookup → Employee List** (`{Id, Value}`) and do **not** carry an email. The email must be resolved against `Employee List`, matching the pattern the app already uses: `LookUp('Employee List', ID = gSelectedRequest.GRAssignedToID.Id).email` (`Src/GoodsReceiptScreen.pa.yaml:241`, `Src/SupplierFollowUpScreen.pa.yaml:475`).

Per request:

| Status | Assignee column | Recipient email |
|---|---|---|
| `Goods Receipt & Acceptance` | `GRAssignedToID` | If `Id` present (not blank and not `0`) → `Email` from `Employee List`; otherwise → `RequesterEmail` |
| `Pending Supplier Follow-up` | `SFU1AssignedToID` | If `Id` present → `Email` from `Employee List`; otherwise → `RequesterEmail` |

`RequesterEmail` is a **Text** column directly on the request, so the fallback branch needs no lookup.

## Build steps (Power Automate portal)

Suggested flow name: `RM Procurement - Daily Goods Receipt Reminder`.

### 1. Trigger — Recurrence
- Frequency: **Week**, Interval: **1**
- On these days: **Monday, Tuesday, Wednesday, Thursday, Friday**
- At these hours: **8**, At these minutes: **0**
- Time zone: **AUS Eastern Standard Time** (handles Melbourne DST)

### 2. Get items — requests awaiting receipt
- Action: **SharePoint → Get items**
- Site: `https://maxbiocare.sharepoint.com/sites/Powerapps`
- List: `RM Procurement Requests`
- **Filter Query** (OData; filter a Choice column via `/Value`):
  ```
  Status/Value eq 'Goods Receipt & Acceptance' or Status/Value eq 'Pending Supplier Follow-up'
  ```
- Top Count: 5000 (enable pagination if needed). Optional `Order By`: `Modified asc`.

### 3. Initialize variables (before the loop)
- `varRecipientEmail` (String)
- `varRecipientName` (String)

### 4. Apply to each — over `value` from Get items

**4a. Determine assignee Id by status** — a **Compose** (`AssigneeId`):
```
if(equals(item()?['Status']?['Value'], 'Goods Receipt & Acceptance'),
   item()?['GRAssignedToID']?['Id'],
   item()?['SFU1AssignedToID']?['Id'])
```
> The lookup Id path can be `item()?['GRAssignedToID']?['Id']` or a flattened `item()?['GRAssignedToIDId']` depending on connector config — confirm with a test run (see Verification) before locking the expression.

**4b. Condition:** `AssigneeId` **is not equal to** blank **AND** `AssigneeId` **is not equal to** `0`.
- **Yes (has assignee):** add **SharePoint → Get item** (List `Employee List`, Id = `AssigneeId`); set `varRecipientEmail` = its `Email`, `varRecipientName` = the request's `GRAssignedToID`/`SFU1AssignedToID` `Value`.
- **No (unassigned):** set `varRecipientEmail` = `item()?['RequesterEmail']`, `varRecipientName` = `item()?['RequesterID']?['Value']`.

**4c. Send an email (V2)** — Office 365 Outlook
- Connection: shared account **`app.admin@maxbiocare.com`** (same as existing flows; pin under "Run only users" if shared).
- **To:** `varRecipientEmail`
- **Subject:** `Reminder: Please receive goods & enter details — {Title}`
- **Body (short):**
  ```
  Hi {varRecipientName},

  The following procurement request is waiting for you to receive the goods
  and enter the goods details (Received Qty, Batch Number, QC Number,
  Expiry Date, RM/PK Code, Link to COA):

  • Request: {Title}
  • Status: {Status Value}
  • Required delivery date: {RequiredDeliveryDate dd/MM/yyyy}

  Please open the request in the Raw Materials Procurement app to complete it:
  {link}

  Thank you.
  ```
  - `{RequiredDeliveryDate}`: `if(empty(item()?['RequiredDeliveryDate']), 'N/A', formatDateTime(item()?['RequiredDeliveryDate'], 'dd/MM/yyyy'))`.
  - `{link}`: simplest is the SharePoint display form:
    `https://maxbiocare.sharepoint.com/sites/Powerapps/Lists/RM%20Procurement%20Requests/DispForm.aspx?ID={ID}`.
    To deep-link into the Power App instead, use the app Play URL plus the request Id (needs Environment Id + App Id).

## Notes & edge cases

- **One email per request.** If a person is responsible for several requests they receive several emails. Grouping into one email per person is possible but more complex; kept 1-per-request for simplicity given the short-email requirement.
- `RequiredDeliveryDate` empty → guarded with `if(empty(...), 'N/A', ...)`.
- `AssigneeId = 0` (assignee cleared by "I will receive") is treated as unassigned → email goes to the requester. Covered by the 4b condition.
- Reminds daily with no overdue threshold — the same email repeats each morning until the request leaves the status.
- Lookup Id path (`['GRAssignedToID']['Id']` vs `['GRAssignedToIDId']`) varies by config — confirm via a test run before locking expressions.

## Verification

1. **Manual test:** in the portal, **Test → Manually** to run without waiting for 08:00.
2. **Inspect Get items:** in run history, confirm `value` only contains requests in the two target statuses; inspect the JSON of `GRAssignedToID`/`SFU1AssignedToID` to lock the correct `Id` path for step 4a.
3. **Test data on SharePoint:** ensure at least three requests: (a) `Goods Receipt & Acceptance` **with** `GRAssignedToID`; (b) `Goods Receipt & Acceptance` **without** an assignee (expect email to `RequesterEmail`); (c) `Pending Supplier Follow-up` **with** `SFU1AssignedToID`. Verify each branch resolves the correct email in run history.
4. **Check the email:** correct recipient, and `Title` / `Status` / delivery date / link render correctly and open the right request.
5. **Check the schedule:** after enabling, confirm the trigger is scheduled for 08:00 AUS Eastern, Monday–Friday only.

## Open questions for setup

- Should the email link be the **SharePoint display form** or a **Power App Play link** (needs Environment Id + App Id)?
- Confirm the sending mailbox is `app.admin@maxbiocare.com`, consistent with existing flows.
