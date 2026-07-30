# Power Automate — Daily Link to COA Completion Reminder Flow

Design/build guide for a **scheduled cloud flow** that sends Procurement a daily digest of outstanding "Link to COA" entries.

> This is a **standalone** Power Automate flow built in the portal. It is **not** called from the Power App and requires **no** `.pa.yaml` change. This document is the build/reference guide only.

## Why

`COALink1` (Goods Receipt, round 1) and `COALink2` (Supplier Follow-up, round 2) on `'RM Procurement Line Items'` are **optional** at submit time on `GoodsReceiptScreen` / `SupplierFollowUpScreen` — the request keeps moving through its normal Status workflow even if a receiver leaves them blank. Completing them later is deliberately **decoupled from the request workflow**: Procurement fills them in via `Src/COACompletionScreen.pa.yaml` (reached via a "Link to COA" button on that request's row in `HomeScreen`), whenever convenient, without blocking or re-routing any request. Nothing currently prompts Procurement to go check which requests need this, so this flow adds a daily digest nudge.

## Behavior (agreed requirements)

- **What counts as outstanding:** a `'RM Procurement Line Items'` row where `(ReceivedQty1 > 0 and COALink1 is blank)` or `(ReceivedQty2 > 0 and COALink2 is blank)`, **excluding** line items whose parent request `Status = "Rejected"` (a rejected request has no further use for a COA link).
- **One digest, not one email per request.** Every run, build a single list of every outstanding request + material + round, and send **one email** to the recipients below. If there are zero outstanding rows, send **no email** that day.
- **Recipients:** every `'RM User'` row with `Role = "Procurement"` (resolved to an email via `Employee List`). Not per-assignee (there is no assignee column for this task) and not Admin — Procurement only, as scoped.
- **Schedule:** 08:30 Melbourne time, Monday–Friday (staggered 30 minutes after the existing `RM Procurement - Daily Goods Receipt Reminder` flow so the two don't compete for review time).

## Query approach

`Status` lives on `'RM Procurement Requests'`, not on the line item, so the "exclude Rejected" check can't be expressed in a single line-item OData filter. Build it as a **nested loop keyed on `RequestIDText`** — the same join key the app itself uses everywhere (see `CLAUDE.md`: `RequestIDText` is used to look up related rows):

1. Get all non-Rejected requests first.
2. For each one, get its outstanding line items (if any) and append a digest line.

This avoids hand-written array-join expressions and stays consistent with how the rest of this app's flows are built.

## Build steps (Power Automate portal)

Suggested flow name: `RM Procurement - Daily COA Completion Reminder`.

### 1. Trigger — Recurrence
- Frequency: **Week**, Interval: **1**
- On these days: **Monday, Tuesday, Wednesday, Thursday, Friday**
- At these hours: **8**, At these minutes: **30**
- Time zone: **AUS Eastern Standard Time**

### 2. Get items — active (non-Rejected) requests
- Action: **SharePoint → Get items**
- Site: `https://maxbiocare.sharepoint.com/sites/Powerapps`
- List: `RM Procurement Requests`
- **Filter Query:** `Status ne 'Rejected'`
- Select columns: `ID, Title` (Status not needed further — already filtered)
- Top Count: 5000 (enable pagination if needed)

### 3. Initialize variable — `varDigestLines` (Array, empty)

### 4. Apply to each — over `value` from step 2
- **Get items** ("Line Items for Request"): List `RM Procurement Line Items`, Filter Query:
  ```
  RequestIDText eq '{items('Apply_to_each')?['ID']}' and ((ReceivedQty1 gt 0 and COALink1 eq null) or (ReceivedQty2 gt 0 and COALink2 eq null))
  ```
  > If a test run shows blank COA links stored as empty string rather than null (same caveat as the existing Goods Receipt reminder flow), extend to `... or COALink1 eq ''` / `... or COALink2 eq ''`.
- **Condition:** `length(outputs('Get_items_-_Line_Items_for_Request')?['body/value'])` is greater than `0`
  - **Yes:** **Apply to each** over that inner `value` →
    **Append to array variable** `varDigestLines`, Value:
    ```
    concat(items('Apply_to_each')?['Title'], ' — ', item()?['MaterialName'], ' — Round ',
        if(and(greater(item()?['ReceivedQty1'], 0), empty(item()?['COALink1'])), '1', '2'))
    ```
  - **No:** do nothing for this request.

### 5. Condition — anything to report?
`length(variables('varDigestLines'))` is greater than `0`
- **No:** end the flow here — no email sent.
- **Yes:** continue to recipients + email.

### 6. Get items — Procurement recipients
- List: `'RM User'`, Filter Query: `Role eq 'Procurement'`

### 7. Initialize variable — `varRecipientEmails` (Array, empty), then **Apply to each** over step 6's `value`:
- **Get item** (List `Employee List`, Id = `item()?['EmployeeID']?['Id']`) — confirm the exact lookup Id path via a test run (`['EmployeeID']['Id']` vs a flattened `['EmployeeIDId']`, same caveat as the existing GR reminder flow).
- **Append to array variable** `varRecipientEmails`, Value: `body('Get_item_-_Employee')?['Email']`

### 8. Send an email (V2) — Office 365 Outlook
- Connection: shared account **`app.admin@maxbiocare.com`** (same as existing flows; pin under "Run only users").
- **To:** `join(variables('varRecipientEmails'), ';')`
- **Subject:** `Daily Reminder: {length(varDigestLines)} material line item(s) awaiting Link to COA`
- **Body:**
  ```
  Hi team,

  The following material line items are still missing a Link to COA
  (Certificate of Analysis). Please open each request in the Raw Materials
  Procurement app and use its "Link to COA" button to complete them:

  {join(variables('varDigestLines'), '<br>')}

  Thank you.
  ```

## Notes & edge cases

- **Digest, not per-request emails.** All outstanding items across all requests are combined into one email per recipient per day, sent to every Procurement-role user — not filtered per-person, since completing a COA link isn't assigned to a specific individual.
- **Procurement role only**, per explicit scope — Admin is not included. If Admin should also receive it, widen step 6's filter to `Role eq 'Procurement' or Role eq 'Admin'`.
- **Rejected requests excluded** entirely via step 2's filter — their line items never appear in the digest even if COA links were left blank.
- **Nested loop cost:** one extra `Get items` call per active request. Fine at this app's typical volume; if request counts grow large, replace with a single `Get items` over Line Items plus a `Filter array` keyed against the active-request ID set instead.
- Reminds daily with no "already nudged" suppression — the same outstanding rows repeat in the digest each morning until someone fills them in via `COACompletionScreen`.

## Verification

1. **Manual test:** in the portal, **Test → Manually** to run without waiting for 08:30.
2. **Test data:** ensure at least: (a) a non-Rejected request with a line item missing `COALink1` (`ReceivedQty1 > 0`); (b) a non-Rejected request with a line item missing `COALink2` (`ReceivedQty2 > 0`); (c) a Rejected request with a similarly-missing line item (expect it **excluded**); (d) a request where everything is filled in (expect no digest line).
3. **Check the digest:** confirm every expected line appears with the correct Round, and no false positives/negatives.
4. **Check recipients:** confirm the `To` list matches the current set of `'RM User'` rows with `Role = "Procurement"`.
5. **Check the zero-outstanding case:** temporarily fill in all test data's COA links and re-run — confirm no email is sent.
6. **Check the schedule:** after enabling, confirm the trigger is scheduled for 08:30 AUS Eastern, Monday–Friday only.

## Open questions for setup

- Confirm the sending mailbox is `app.admin@maxbiocare.com`, consistent with existing flows.
- Preferred link in the email body: none (as drafted, just points users to the app by name) or a direct Power App Play link (needs Environment Id + App Id)?
