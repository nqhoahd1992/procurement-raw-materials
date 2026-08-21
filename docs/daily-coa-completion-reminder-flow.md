# Power Automate — Daily Link to COA Completion Reminder Flow

Reference for the **scheduled cloud flow** that sends Procurement a daily digest of requests still missing a "Link to COA".

Live flow name in the portal: **`RM Procurement - Update COA Reminder for Procurement`**.

> This is a **standalone** Power Automate flow built in the portal. It is **not** called from the Power App and requires **no** `.pa.yaml` change. This document mirrors the flow as actually built — action names below are the real ones, so `outputs('…')` / `body('…')` references can be copied verbatim.

## Why

`COALink` on `'RM Procurement Receipt Rounds'` is **required** at submit time on `GoodsReceiptScreen` / `SupplierFollowUpScreen`, so rounds recorded from that change onward are never outstanding — this flow now only ever surfaces rows created while it was still optional, plus any row whose link is later cleared. Fixing one is deliberately **decoupled from the request workflow**: Procurement edits it via `Src/ReceiptAmendmentScreen.pa.yaml` (reached via the "Amend Receipt Data" button on that request's row in `HomeScreen`), whenever convenient, without blocking or re-routing any request. Nothing in the app prompts Procurement to go check which requests need this, so this flow is the nudge.

## What it reports

- **What counts as outstanding:** a `'RM Procurement Receipt Rounds'` row with `COAMissing = Yes`. Nothing more — a receipt-round row only exists for a material actually received in that round (`ReceivedQty > 0`), so the old `ReceivedQty1 gt 0` / `ReceivedQty2 gt 0` guards are gone.
- **Granularity: request level, not line-item level.** Outstanding rows are collapsed to their parent request, so the email lists *requests* pending COA, not individual material/round rows. Procurement can't act on a single row from the email anyway — they open the request and `ReceiptAmendmentScreen` lists every receipt round for it. This is a deliberate change from the original design (which listed `<request> — <material> — Round <n>` per row); the trade-off is that the email doesn't show *how many* rows a request is missing.

> ⚠ **Since the parent/delivery split, "request" here means a *delivery* row, not the Requester's original request.** Receipt-round rows are written against the **delivery** (`RequestIDNum` = the delivery's ID), so `RequestID.Value` resolves to the delivery's Title — `<parent title> - #<n>`. That string still names the parent, so the email stays readable, but the `union()` dedupe in step 5 is **per delivery**: one original request with three COA-deficient deliveries produces three lines, not one. That is arguably the more useful granularity (each delivery has its own COA paperwork), but it contradicts the "request level" claim above, so read this section as *delivery level*. If you want true parent-level grouping, project the new `ParentRequestID` (Number) column in the step-4 `Select` and dedupe on that instead. See `CLAUDE.md`'s "Two request types" section.
- **Recipients:** every active `'RM User'` row with `Role = "Procurement"`, resolved to an email via `Employee List`. Not per-assignee (there is no assignee column for this task) and not Admin — Procurement only, as scoped. One email is sent **per recipient** (inside `Apply to each`), so a run sends N emails for N Procurement users.
- **Nothing outstanding → no email.** Guarded by a Condition; see step 6.

## Why `COAMissing` and not `COALink eq null`

`COALink` is a Text (URL) column. Filtering blank text is the thing this schema deliberately avoids:

- In Power Apps, `Filter(…, IsBlank(COALink))` is non-delegable — the SharePoint connector never delegates `IsBlank()` on a text column, and no formula rewrite clears the warning.
- In a flow, `COALink eq null` is unreliable because SharePoint may store an empty string rather than null, and `COALink eq 0` is invalid outright (text compared to a number → OData error).

`COAMissing` is a Yes/No column written locally at the same points the link is written (`GoodsReceiptScreen.btnSubmit_GR`, `SupplierFollowUpScreen.btnSubmitStep1_SFU` set `COAMissing: IsBlank(re.COALink)`; `COACompletionScreen`'s save clears it). **`COAMissing = 1` means the COA is still missing** — the filter is `eq 1`, not `eq 0`. See `docs/sharepoint-schema.md` for the column notes.

## SharePoint prerequisites

Two **single-column indexes** on `'RM Procurement Receipt Rounds'` (List settings → Indexed columns), both with Secondary column `(none)`:

1. `COAMissing` — used by this flow's filter.
2. `RequestIDNum` — used by the app (`ReceiptAmendmentScreen`, `HomeScreen`). This superseded `RequestIDText`: the app now joins on the delegable `RequestIDNum` (Number) column instead of `RequestIDText = Text(id)`, which was never actually delegable — see "Pending manual SharePoint work for delegable request-ID lookups" in `CLAUDE.md`. Re-point this index at `RequestIDNum` once that column exists and is backfilled.

No compound index is needed: every query filters on one of the two, and the app's `RequestIDNum = … && COAMissing` narrows on the indexed number column first.

This list grows one row per (line item × receipt round), so it reaches 5,000 items faster than the others — and past 5,000 SharePoint **refuses to create new indexes**, leaving the queries permanently stuck against the list-view threshold. Create both indexes while the list is small.

## Build steps (Power Automate portal)

### 1. Trigger — Recurrence
- Frequency: **Week**, Interval: **1**
- On these days: **Monday–Friday**
- At these hours: **8**, At these minutes: **30**
- Time zone: **AUS Eastern Standard Time**

Staggered 30 minutes after `RM Procurement - Daily Goods Receipt Reminder` so the two don't compete for review time.

### 2. Initialize variable — `varConfig`
Object holding shared config; this flow reads `varConfig?['appLink']` for the email's CTA button. **The `appLink` key must exist** — the email uses the safe-navigation form `variables('varConfig')?['appLink']`, so a missing key yields an empty string and the button silently links nowhere while the run still reports Succeeded.

### 3. `Get items COA missing` — SharePoint → Get items
- Site: `https://maxbiocare.sharepoint.com/sites/Powerapps`
- List: **`RM Procurement Receipt Rounds`**
- **Filter Query:** `COAMissing eq 1`
- **Top Count:** `5000`
- Settings tab → **Pagination: On**, Threshold `5000`
- Connection: `app.admin@maxbiocare.com`

Top Count and Pagination are both required: left blank, SharePoint Get items returns only the **first 100 items** and truncates silently — the digest would just be missing requests, with no error anywhere. Top Count caps a single request (5000 is SharePoint's maximum); the Pagination threshold is how many items the connector will accumulate across pages. To go past 5,000 later, raise the threshold and leave Top Count at 5000.

### 4. `Select` — project to parent request
```json
{
  "from": "@outputs('Get_items_COA_missing')?['body/value']",
  "select": {
    "ID": "@item()?['RequestID']?['Id']",
    "Title": "@item()?['RequestID']?['Value']"
  }
}
```
`RequestID` is a Lookup to `'RM Procurement Requests'` (`{Id, Value}` where Value is the request Title), so the request's name is available here — no second `Get items` against the Requests list.

> **Confirm this path after any connector change.** If the connector ever returns the lookup flattened (`RequestIDId` / `RequestIDValue`), both fields become `null`, step 5 collapses them into a single `{ID: null, Title: null}` entry, and the email sends one meaningless line — with the run still Succeeded. If that happens, `RequestIDNum` (plain number, always present once backfilled) is the robust source for `ID`. **Not `RequestIDText`** — that column is deprecated and no longer written by the app; don't build a new fallback on a column being phased out (see `CLAUDE.md`).

### 5. `Deduplicate` — Compose
```
@union(body('Select'), body('Select'))
```
The standard array-dedupe idiom. It is correct here because `Select` emits only `{ID, Title}`, so the many outstanding rows of one request collapse to one entry.

### 6. `Select Format` — one HTML card per request
```
@concat('<div style="padding:11px 15px;margin-bottom:8px;background-color:#fdf6f7;border-left:3px solid #ad1f39;border-radius:6px;font-size:13.5px;color:#2c3e50;font-family:''Segoe UI'',Calibri,Arial,sans-serif;"><strong style="color:#ad1f39;">#', item()?['ID'], '</strong>&nbsp;&nbsp;', item()?['Title'], '</div>')
```
Source: `@outputs('Deduplicate')`.

Single quotes inside the HTML are **doubled** (`''Segoe UI''`) — that is the expression language's escape, not a typo. The red left border acts as the bullet, so no `•` character is needed.

### 7. `Compose Email List` — join the cards
```
@join(body('Select_Format'), '')
```
The separator is intentionally **empty**: each element is already a complete `<div>` block. It must stay empty *only* while step 6 emits HTML — if step 6 is ever reverted to plain text (`'• ID: n - Title'`), this separator must become `'<br>'`, or every line runs together into one blob.

### 8. Condition — anything to report?
```
@greater(length(outputs('Deduplicate')), 0)
```
- **True:** steps 9–10.
- **False:** nothing — the flow ends Succeeded without sending.

Without this guard the flow mails a `0 Request(s) Pending COA` subject with an empty list every weekday, which is the fastest way to train the recipients to filter the whole thing out.

### 9. `Get Procurement Staffs` — SharePoint → Get items
- List: `'RM User'`
- **Filter Query:** `Role eq 'Procurement' and IsActive eq 1`

`IsActive` matters: the app filters on it in every people picker, so a deactivated `'RM User'` row must not keep receiving a daily reminder for work they can no longer do.

### 10. `Apply to each` over step 9's `value`
- **`Get item`** — List `Employee List`, Id = `item()?['EmployeeID']?['Id']`.
- **`Send an email (V2)`** — Office 365 Outlook, connection `app.admin@maxbiocare.com` (pinned under "Run only users").
  - **To:** `@outputs('Get_item')?['body/email']`
  - **Subject:** `[MaxBiocare Procurement] COA Reminder – @{length(outputs('Deduplicate'))} Request(s) Pending COA`
  - **Body:** branded HTML table (MaxBiocare red `#ad1f39`), with `@{outputs('Compose_Email_List')}` injected as the request list and a CTA button pointing at `@{variables('varConfig')?['appLink']}`.

Two notes on the email HTML: every gradient is preceded by a plain `background-color` so Outlook desktop (Word rendering engine) degrades to a flat colour instead of white, and `border-radius` is simply ignored there — both intentional. `Employee List`'s email column is referenced as `body/email`; the app reads it as both `Email` and `.email` because Power Fx is case-insensitive, but **Flow is not** — if `To` comes back empty, check the column's internal name.

## Notes & edge cases

- **Rejected requests are not excluded.** `'RM Procurement Receipt Rounds'` has no `Status` column, and a request is only ever Rejected at the Manager / Executive / Procurement stage — before any receipt round exists — so the set is empty in practice and the check was deliberately skipped. Residual case: a request rejected **manually in SharePoint** after goods receipt would be reminded about forever, while `HomeScreen`'s "Link to COA" button hides for Rejected requests, so Procurement couldn't act from the app. Fix that by hand: clear `COAMissing` on those rows.
- **`COAMissing` is only maintained by the app.** Someone filling `COALink` directly in SharePoint leaves the flag at `1`, and that request keeps appearing every morning. Only `ReceiptAmendmentScreen`'s save recomputes it (as `IsBlank(NewCOALink)`).
- **No "already nudged" suppression.** The same requests repeat in the digest each morning until someone completes them.
- **Per-recipient emails.** The send sits inside `Apply to each`, so each Procurement user gets their own copy (rather than one email with a joined `To` list). Cost is one email per recipient per run.
- **Row counts are not shown.** A request missing 1 COA and one missing 20 render identically. Adding a count would need an `Apply to each` plus a variable — inside a nested `filter()` the inner `item()` shadows the outer element, so there is no compact expression for it. Judged not worth the complexity, since opening the request shows everything.

## Verification

1. **Manual test:** **Test → Manually** in the portal, no need to wait for 08:30.
2. **Check `Select`'s output first.** Every entry must have a real `ID` and `Title`. A single `{ID: null, Title: null}` entry means the lookup path broke — see the warning in step 4.
3. **Check `Get items COA missing`'s output count** against a temporary SharePoint view filtered `COAMissing = Yes`. Exactly 100 rows is the tell-tale sign that Top Count / Pagination isn't applied.
4. **Test data:** (a) a request with at least one receipt-round row where `COAMissing = Yes` → expect one card in the email; (b) two outstanding rows on the **same** request → expect still **one** card (dedupe working); (c) everything filled in → expect **no email at all** (Condition working).
5. **Check recipients:** `To` matches the current active `Role = "Procurement"` rows, and the CTA button's href is a real URL (not empty).
6. **Render check:** open the email in Outlook desktop as well as web — the cards should show the red left border and the CTA should be a solid red button.
