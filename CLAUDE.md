# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Max Biocare Procurement Procedure** — a Power Apps Canvas App (DesktopOrTablet, 1366×768) that manages the end-to-end procurement request workflow. The Power Fx source lives in `.pa.yaml` files.

This folder keeps **only the screen logic + UI** (`Src/`). The generated metadata that the full unpacked `.msapp` would contain (`Controls/`, `References/`, `Resources/`, `Properties.json`, `Header.json`, `AppCheckerResult.sarif`) has been **intentionally removed** — the workflow here is to read/edit Power Fx and paste it into **Power Apps Studio in the browser**, not to repack a `.msapp`. Because of that, this folder can no longer be packed with `pac canvas pack`; re-export from Power Apps if you ever need the full package again.

> Note: the sibling working directory `../ci_cd` is a **separate** Node.js/Docker project, unrelated to this Canvas App. Don't conflate the two.

## Layout

- `Src/*.pa.yaml` — one file per screen (the actual logic + UI). Screen order in `Src/_EditorState.pa.yaml`.
- `Src/App.pa.yaml` — `App.OnStart`: resolves the signed-in user and sets global state.

## Backend & connectors

All data lives in **SharePoint Online** (`maxbiocare.sharepoint.com/sites/Powerapps`). Lists:
- `Procurement_Requests` — the central request record (status, approvers, costs, invoice, goods-receipt, follow-up fields).
- `Procurement_User` — maps an employee to a `Role` (Requester / Manager / Executive / Procurement / Accounting / Admin).
- `Procurement_ApprovalLog` — manager/executive decisions (StepNumber 2 = Manager, 3 = Executive).
- `Procurement_ExecutionLog` — procurement/goods-receipt/follow-up step records (StepNumber 1, 3, 4, 5).
- `Procurement_InvoiceData`, `Suppliers`, `Employee List`.
- `Project_List` — cross-app project list (owned by the sibling `project-list` app, same site). Read-only here: the `ddProject_1` picker on `RequestFormScreen`, the project-name label on `RequestDetailScreen`, and the "Project Information" panel on `ManagerReviewScreen` / `ExecutiveApprovalScreen` (`ProjectID`, `ProjectDescription`, `BudgetAmount`, `Currency`). `Procurement_InvoiceData` is read a second way by that same panel — `Filter(Procurement_InvoiceData, ProjectID = gSelectedRequest.ProjectID)`, summed by `Currency` over `TotalAmount` — to show the project's Actual Cost.

Full column-level schema (types, choice values, join keys): `docs/sharepoint-schema.md`.

**Power Automate** flows called from Power Fx:

Invoice flows (called from `ProcurementExecutionScreen` and `InvoiceSubmissionScreen`):
- `Parse_Invoice.Run(invoiceUrl, requestId)` — AI invoice extraction.
- `Submit_Invoice.Run(...)` — 18 positional args; writes parsed invoice data. Param 14 is the invoice **Description** field value, param 17 is the source app name (`"Procurement App"`), param 18 is `gSelectedRequest.ProjectID` — see `docs/sharepoint-schema.md` for the full param table.

Assignment / status notification flow (called from `GoodsReceiptScreen`, `SupplierFollowUpScreen`, `ExecutiveApprovalScreen`, `ExecutivePaymentScreen`):
- `Procurement_Notify_Receipt_Assignee.Run(assigneeEmail, assigneeName, requestTitle, requestId, notificationType, deliveryDate, category, appName)`
  - `notificationType = "GoodsReceipt"` — notify new assignee they must perform Goods Receipt & Acceptance.
  - `notificationType = "SupplierFollowUp"` — notify new assignee they must perform Supplier Follow-up Round 2.
  - `notificationType = "Unassigned"` — notify previous assignee they no longer need to act (email only, no Adaptive Card).
  - `notificationType = "ExecutivePayment"` — notify the approving Executive (`gCurrentEmployee`) they must process payment via `ExecutivePaymentScreen`; called from `btnSubmitExec`'s decision-submit handler when `isExecutivePayment = true`.
  - `notificationType = "ProcurementExecution"` — notify the Procurement team a request is ready at `Pending Procurement`; called from `ExecutivePaymentScreen`'s `formExecutivePayment.OnSuccess`. `assigneeEmail`/`assigneeName` are passed as generic placeholder values (`"procurement@maxbiocare.com"` / `"Procurement Team"`) — fan-out to all Procurement-role users is handled inside the flow, not per-request.
  - Called on `btnSaveAssignment_GR` / `btnSaveAssignment_SFU` (assign new person) and `btnIWillReceive_GR` / `btnIWillReceive_SFU` (requester takes back the task).
  - Flow sends Outlook email + Teams Adaptive Card for assignment types; email only for Unassigned.
  - Connection: `app.admin@maxbiocare.com` pinned in "Run only users" — not invoker-provided.

**Important:** SharePoint system column internal names are English (site locale = en) — use `Title` for the title field and `Attachments` for attachments. Custom columns are also English (`Status`, `EstimatedCost`, `ManagerApproverID`, etc.).

## Global state (set in App.OnStart)

- `gCurrentEmployee` — row from `Employee List` matched by `User().Email`.
- `gCurrentUser` / `gUserRole` — row from `Procurement_User`; defaults to role `"Requester"` if not found.
- `gIsSpecialRole` — true for Manager/Executive/Procurement/Accounting/Admin (drives toolbar/filter visibility).
- `gSelectedRequest` — the request being viewed/acted on (set before `Navigate`).
- `gParsingInvoice`, `gHasInvoiceResult`, `gInvoiceResult`, `gShowRejectReason`.
- `colLogDetail` / `gShowLogDetail` / `gLogDetailTitle` — `RequestDetailScreen`'s **step-details dialog** (`rectLogDetailOverlay_RD` / `conLogDetailDialog_RD`, the **last two screen-level children**). Deliberately generic: `colLogDetail` is a `{FieldLabel, FieldValue}` pair list, so **one** dialog serves all three history tables. Each row's "Details" button (`btnGRLogDetail` / `btnApprovalLogDetail` / `btnExecLogDetail`) does `ClearCollect(colLogDetail, {…}, {…}, …); RemoveIf(colLogDetail, IsBlank(FieldValue))` — every candidate field is listed unconditionally and the blanks dropped afterwards, which is why each value is wrapped in `Coalesce(…, "")` (`IsBlank("")` is true, so `RemoveIf` catches them). It exists because a Power Apps gallery can't have per-row heights: the table row truncates notes/remarks/conditions on one line with `Wrap: =false`, and this dialog is the only way to read the full text.
- `colReceiptPhotos` / `gShowReceiptPhotos` / `gPhotoRoundLabel` — the **receipt-attachments dialog** on `RequestDetailScreen` (`rectPhotoOverlay_RD` / `conPhotoDialog_RD`). Each Goods Receipt / Supplier Follow-up row's "Attachments" button does `ClearCollect(colReceiptPhotos, LookUp('Procurement Receipt Rounds', ID = ThisItem.ID).Attachments)` and flips `gShowReceiptPhotos`; the button is gated on the matching `gGRHasAttachments` / `gSFU1HasAttachments` / `gSFU2HasAttachments` flag computed in `OnVisible`. The dialog lists file names, each one `Launch(…AbsoluteUri, {}, LaunchTarget.New)`. **Attachments are only reachable on a record fetched by `LookUp`** — the column is not part of the row scope of a `ForAll`/`Filter` over a SharePoint list, so don't try to project it into a collection.
- **`RequestDetailScreen` surfaces**: beige `RGBA(241, 239, 232, 1)` is *the request as it stands* and is used by `rowInfoBlock` only — which is why `rowCreditNote` lives there, next to the other document links. White `RGBA(255, 255, 255, 1)` is *history*: the three sibling sections `rowGRSection` / `rowApprovalSection` / `rowExecutionSection`. All three share one table style (32px header filled `RGBA(232, 230, 221, 1)` with size-9 Semibold `RGBA(107, 105, 98, 1)` labels; plain white rows with a 1px `RGBA(232, 230, 221, 1)` hairline as the **last child of the row template**; no zebra striping) and one row geometry (`TemplateSize: =52`, body 51 with `PaddingTop`/`Bottom 4` and `LayoutGap 10`, a fixed **96px two-line timestamp column** then a `FillPortions: =1` column holding a 24px cells row and a 15px notes line). Section height is `108 + CountRows(...) * 52`, or `134 + …` for `rowGRSection` which carries an extra summary line — change the row height and all four numbers move together. Semantic colors are reserved for outcomes (green `15,110,86` accepted/approved, red `163,45,45` rejected, amber `133,79,11` needs attention); blue `56,96,178` is links; purple `83,74,183` is the app header, dialog titles and primary buttons. This mirrors the sibling `procurement-raw-materials` app control-for-control — keep the two in sync.
  - **`rowGRSection` reads the request, not the log.** Its gallery iterates `Filter(colExecutionLog, StepNumber >= 3 && StepNumber <= 5)` purely to know **which** steps happened (and to reach each step's attachments by log `ID`); every displayed value — receiver, receipt date, receipt status, acceptance decision, remarks — is pulled off `gSelectedRequest` with a `Switch` on `StepNumber`, because in this app the receipt data lives in request columns (`GoodsReceiptBy`, `FollowUpReceiptStatus`, …), not on the log row. That is the structural difference from the raw-materials app, where unbounded receipt rounds forced the same data onto per-round rows.
- `gShowQuickView` / `gQuickViewRequest` — `HomeScreen`'s **Quick View** lightbox (`rectQuickViewOverlay_HS` / `conQuickViewDialog_HS`, the **last screen-level children before the status-filter dialog** — z-order in Canvas is child order). Each gallery row's `btnQuickView` does `Set(gQuickViewRequest, ThisItem); Set(gShowQuickView, true)` — `gQuickViewRequest` is now the **whole SharePoint row** (not a curated literal), so the dialog and its footer action button can read any field off it. It's still a **separate var from `gSelectedRequest`** so opening the popup doesn't mutate the navigation target; "Open Full Detail" is what copies it into `gSelectedRequest` before navigating. Shows Title, Category/ProcurementType, a Project/Required-Delivery-Date/Requested-by row (`rowQVIdent_HS`, project name resolved via `LookUp(Project_List, ProjectID = gQuickViewRequest.ProjectID).Title`), ProcurementDescription (scrolling label, the only `FillPortions: =1` child), `RequesterInvoiceURL` (gated on `ProcurementType = "Invoice Supplied"`, rendered as "Open invoice ↗" with the real URL only in `Tooltip`) and `OfficialInvoiceLink` (same "Open official invoice ↗" treatment). The footer carries a primary **`btnQVDoWork_HS`** action button — the same per-status `Switch`/`Visible` logic as the gallery row's `btnDoWork` (see below), just keyed off `gQuickViewRequest` instead of `ThisItem` — plus "Open Full Detail" and "Close". `gShowQuickView` is reset to `false` in `HomeScreen.OnVisible` and seeded in `App.OnStart`. This popup replaced the per-row `lblRequestDesc` label, which is why the gallery row is 100px, not 130 (`galRequests.TemplateSize`, `rowRequestContent.Height` 100, `rectRowHitArea.Height` 82, `rectSeparator.Y` 99 — change all four together). The layout mirrors the sibling `procurement-raw-materials` app's `HomeScreen` control-for-control (minus that app's delivery/line-item concepts, which don't exist here); keep the two in sync when either changes.
- **Status filter is a multi-select dropdown, not a chip bar.** `gShowStatusFilter` / `colStatusFilterOptions_HS` (the fixed list of this app's statuses, seeded in `HomeScreen.OnVisible`) / `colStatusFilterDraft` (in-progress checkbox state while the dialog `conStatusFilterDialog_HS` is open) / `colStatusFilterChecked` (the applied filter — empty means "no filter", i.e. show all). `btnStatusFilterDropdown_HS` in `rowToolbar` seeds the draft from whatever's currently applied and opens the dialog; `btnStatusFilterApply_HS` commits the draft into `colStatusFilterChecked`, `btnStatusFilterCancel_HS` discards it. `galRequests.Items` filters on `IsEmpty(colStatusFilterChecked) || Status.Value in colStatusFilterChecked`. Each status row in the dialog shows a live count from the same per-role `Switch` used by `galRequests.Items`, just counted per status instead of filtered by the checked set — keep that `Switch` copy in sync with the gallery's. Mirrors `procurement-raw-materials`' `HomeScreen`.
- **`btnDoWork`** (gallery row, `rowActions`, first action button) is a single primary "do the next thing" button: a `Switch` on `Status.Value` that navigates straight to the screen that status is waiting on (`ManagerReviewScreen`, `ExecutiveApprovalScreen`/`ExecutivePaymentScreen` per `isExecutivePayment`, `ProcurementExecutionScreen`, `InvoiceSubmissionScreen`, `GoodsReceiptScreen`, `SupplierFollowUpScreen`/`ProcurementFollowUpScreen` per `LatestReceiptDecision`, `AccountingScreen`), with a matching `Visible` `Switch` gating each branch on the same role/assignment checks used elsewhere in the app (manager-approver match, GR/SFU assignee match, Procurement/Admin, etc.). It sits alongside — not instead of — the existing `btnProcessInvoice`/`btnSubmitInvoice` buttons, which still own the two invoice-specific paths (`InvoiceMode = "Deferred"`/blank-URL "Remind Requester" and the Requester's own `"ViaRequester"` submit); `btnDoWork`'s `"Pending Invoice"` branch is therefore gated to exclude both those modes so the two don't double up. Mirrors `procurement-raw-materials`' `HomeScreen` (minus its delivery/receipt-amendment branches, which don't apply here).
  - **Because `btnDoWork` (and the Quick View's `btnQVDoWork_HS`) can now land on `ManagerReviewScreen`, `ExecutiveApprovalScreen`, `ExecutivePaymentScreen`, `ProcurementExecutionScreen`, `GoodsReceiptScreen`, `SupplierFollowUpScreen`, `ProcurementFollowUpScreen` and `AccountingScreen` directly from `HomeScreen`, those eight screens' `btnBack*` now use `Back()` instead of a hardcoded `Navigate(RequestDetailScreen)`** — the old hardcoded target assumed every entry came through `RequestDetailScreen`, which is no longer true. Screens still reached only one way keep a hardcoded target: `RequestFormScreen`/`RequesterInvoiceScreen`/`RequestDetailScreen` still `Navigate(HomeScreen)` (their only entry point), and `InvoiceSubmissionScreen` keeps its `If(gInvoiceFromPFU, Navigate(ProcurementFollowUpScreen), Navigate(HomeScreen))` special-case rather than switching to `Back()`, since it has to know specifically whether to return to `ProcurementFollowUpScreen`. Mirrors `procurement-raw-materials`, which made the same `Back()` switch for the same reason.
- `gPendingAttachments` / `gPendingRequestFiles` / `gPendingInvoiceName` — `RequestFormScreen`'s two attachment controls and the bookkeeping that lets them share one SharePoint column. The screen has **two** cards bound to `Procurement_Requests.{Attachments}`, in two forms: `Form1`/`DataCardValue15` (the invoice, `MaxAttachments: =1`, only visible and required when `ProcurementType = "Invoice Supplied"`) and `formRequestFiles`/`attRequestFiles` (**Requirement Files** — optional, `MaxAttachments: =10`, always visible). Both keep `Default: =ThisItem.Attachments` with `Items: =If(IsBlank(Parent.Default), gPending…, Parent.Default)`: the request doesn't exist while the form is being filled, so each control is in `NewForm` mode, and the `EditForm`+`SubmitForm` that runs after the request row is created would otherwise wipe the user's picks — the `gPending…` var carries them across that reset. The request row itself is created by `Patch`, not by either form; both forms are pure attachment vehicles.
  - **Submit is chained, never parallel.** `btnSubmit_1` submits `formRequestFiles`, and *its* `OnSuccess` runs `EditForm(Form1); SubmitForm(Form1)`; `Form1.OnSuccess` is the terminal step (patches `RequesterInvoiceURL`, resets every input, navigates). Two `SubmitForm`s fired at once against the same item race each other. When there are no requirement files the chain is skipped and `Form1` is submitted directly — `Form1` is always submitted, even on the non-invoice path where it sits inside a hidden container, because its `OnSuccess` is what navigates home.
  - `gPendingInvoiceName` exists because `Form1` now submits **after** the requirement files are already on the item. The old `First(Form1.LastSubmit.Attachments).AbsoluteUri` no longer picks the invoice — and per `docs/powerapps-form-attachment-pattern.md` rule 3 it was never safe anyway, since SharePoint returns `LastSubmit.Attachments` in **alphabetical** order. The name is captured from `Last(DataCardValue15.Attachments).Name` **before** submit and the URL composed the way every other upload does: `gSharePointAttachmentBase & Text(ID) & "/" & EncodeUrl(name)`.
- `colRequirementFiles` — the "Requirement Files (Requester)" list, on `RequestDetailScreen` (`rowRequirementFiles_RD` / `galRequirementFiles_RD`, just above `rowInvoiceLink`), `InvoiceSubmissionScreen` (`rowReqRequirementFiles_ISS`) and `ProcurementExecutionScreen` (`rowReqRequirementFiles_PE`). `{FileName, FileURL}`, rebuilt by **each** screen's `OnVisible` by splitting `gSelectedRequest.RequirementFiles` on `|` and composing each URL from `gSharePointAttachmentBase`; seeded with a literal in `App.OnStart` for its design-time schema. It is a global keyed to whatever `gSelectedRequest` currently is, so any new screen that shows it must rebuild it on entry rather than trust what the previous screen left behind. The row's `Visible`/`Height` and the `CountRows(colRequirementFiles)` term in `rowInfoBlock.LayoutMinHeight` must stay in sync — that container sizes itself by summing its visible children.
  - **Why a `RequirementFiles` name-list column exists.** `Procurement_Requests.Attachments` is one undifferentiated pile shared by every upload in the app, so nothing about a file says which step produced it. Every other upload writes a pointer into its own column at write time (`RequesterInvoiceURL`, `RemittanceURL`, `OfficialInvoiceLink`, `CreditNote`, …); `RequirementFiles` is the same trick for the one upload that can be several files at once — `Concat(attRequestFiles.Attachments, Name, "|")`, written by the create `Patch`. `|` is safe as a delimiter because SharePoint forbids it in file names. Don't try to identify these files at read time instead: by position (SharePoint doesn't guarantee attachment order) or by subtracting the known URL columns (compares SharePoint's own `AbsoluteUri` against URLs this app composed with `EncodeUrl`, so any encoding difference mislabels a file silently).
  - Mirrors the same feature in the sibling `procurement-raw-materials` app — keep the two in sync if either changes.
- `gProjectInfo` / `colProjectInvoices` / `gShowRateInfo` — set in `ManagerReviewScreen.OnVisible` and `ExecutiveApprovalScreen.OnVisible` (**not** in `App.OnStart`), backing the "Project Information" panel: `gProjectInfo` is the `LookUp(Project_List, ProjectID = gSelectedRequest.ProjectID)` row (blank when the request isn't tied to a project), `colProjectInvoices` the matching `Procurement_InvoiceData` rows, `gShowRateInfo` the "Currency Exchange Reference" dialog toggle (reset to `false` on every screen entry). The AUD conversion rates behind the panel's Grand Total are hardcoded in a `Switch(Upper(Currency), ...)` inside the `lblProjectGrandTotal_MR`/`_EA` labels and mirrored as plain text in the dialog — change both together, and keep them in sync with the same table in `project-list`'s `ViewProjectScreen` and in the sibling `procurement-raw-materials` app. The **same rate table** is now also duplicated in `RequestFormScreen`'s 5,000-AUD-equivalent escalation check (6 occurrences) and `ExecutiveApprovalScreen`'s manager-skipped-reason label — when rates change, update all of these together too.

If `gCurrentEmployee.ID` is blank, the user sees an "account not found" message and no UI — the app is membership-gated by the `Employee List`.

## The workflow — this is the core domain model

Requests move through a `Status` choice field. Each screen patches `Procurement_Requests.Status` and writes a log row. The status string is also the value of the `HomeScreen` filter buttons and the gallery color coding.

```
RequestFormScreen (Requester submits)
   │  Auto-skip to Executive if: PurchaseAccordance = "Urgent"/"Unplanned",
   │  OR EstimatedCost > 5,000-AUD-equivalent (any currency, converted via the same
   │  hardcoded FX `Switch(Upper(Currency), ...)` table used elsewhere)  → sets SkippedManagerReview
   ▼
Pending Manager ──(ManagerReviewScreen)──┐
   │ "Approved (within budget)" AND Currency = "AUD" AND EstimatedCost <= 10000 → Pending Procurement
   │ otherwise                                                                  → Pending Executive
   ▼
Pending Executive ──(ExecutiveApprovalScreen)──┐
   │ Reject → Rejected
   │ Approve / Approve with conditions (ConditionsText), then:
   │     ProcurementType = "Invoice Supplied" AND (Currency <> "AUD" OR EstimatedCost > 10000)
   │       → stays "Pending Executive" + isExecutivePayment = true
   │         (shown as "Pending Payment From Executive" in UI only —
   │         real Status string is unchanged) → notify Executive → ExecutivePaymentScreen
   │     otherwise                                  → Pending Procurement
   ▼
Pending Procurement ──(ProcurementExecutionScreen)──┐
   │ Parse_Invoice / Submit_Invoice; → Pending Accounting  (or Rejected)
   ▼
Pending Accounting ──(AccountingScreen)── → Goods Receipt & Acceptance
   ▼
Goods Receipt & Acceptance ──(GoodsReceiptScreen)──┐
   │ Accepted                    → Completed
   │ Rejected                    → Rejected
   │ Requires Supplier Follow-up → Pending Supplier Follow-up
   ▼
Pending Supplier Follow-up ── THE RECEIVING LOOP, capped at 2 rounds ──┐
   │
   │  ┌─ SupplierFollowUpScreen — Requester or SFU1AssignedToID, ExecutionLog Step 4, RoundNumber = N
   │  │     shown when LatestReceiptDecision = "Requires Supplier Follow-up"; the round being
   │  │     entered is ReceiptRoundCount + 1. Receiver is re-chosen every round.
   │  │     Writes ReceiptRoundCount = N, LatestReceiptDecision = <decision>, clears SFU1AssignedToID.
   │  │
   │  │     Round 2 offers only two decisions — see the cap below — so it always ends here:
   │  │     Accepted                    → Fulfillment = "Fulfilled" → Pending Invoice /
   │  │                                   Pending Accounting  (receiving ends)
   │  │     Accepted with Adjustment    → stays Pending Supplier Follow-up
   │  │                                   → hands over to Procurement ↓
   │  └─
   │
   │  ┌─ ProcurementFollowUpScreen — the PROCUREMENT CLOSE-OUT (see below; not a round)
   │  │     Procurement/Admin, ExecutionLog Step 5
   │  │     Runs at most ONCE per request, and only on the "Accepted with Adjustment" branch.
   │  │     shown when Status = "Pending Supplier Follow-up" && LatestReceiptDecision = "Accepted with Adjustment"
   │  │     BLOCKED until the Official Invoice data exists (InvoiceSubmitted +
   │  │     OfficialInvoiceLink) — Procurement must submit it first via
   │  │     InvoiceSubmissionScreen, see below.
   │  │     Requires Remarks + a Credit Note attachment. Writes CreditNote,
   │  │     Fulfillment = "Fulfilled with Adjustment", Status = "Pending Accounting",
   │  │     and ends the receiving process.
   │  └─
   ▼
Completed
```

**Deliveries are capped at 2 rounds.** `ddAcceptanceDecision_GR` (round 1) offers `Accepted` · `Rejected` · `Requires Supplier Follow-up`, but `ddAcceptanceDecision2_SFU` (round 2) offers only `Accepted` · `Accepted with Adjustment` — there is no option that opens a round 3, so receiving always terminates at round 2. The data model itself is still unbounded (`ReceiptRoundCount` is a plain number, one `'Procurement Receipt Rounds'` row per round), so raising the cap later means adding `Requires Supplier Follow-up` back to that one dropdown and relaxing the two `< 2` guards below — no schema change. `gSFURoundOpen` and `RequestDetailScreen`'s "Record Goods Receipt Round N →" button both add `ReceiptRoundCount < 2` on top of the decision check: the dropdown alone already makes round 3 unreachable, so those guards exist only to stop a hand-edited SharePoint value from re-opening the loop.

### The two Procurement work screens carry the whole requester submission

`InvoiceSubmissionScreen` and `ProcurementExecutionScreen` both open with a beige `rowRequestInfo_*` block listing **every field the Requester filled in on `RequestFormScreen`**, as the first child of their `conScrollable_*`. Same shape on both: a 32px header row (bold "Request Details" + status badge), then a `conRequestInfoFields_*` column of 60px four-up field rows, a 96px description, the `colRequirementFiles` links, and conditional link rows. Both screens scroll the block rather than pinning it — `conScrollable_*` sits at `Y: =60` with `Height: =Parent.Height - 120` (app header 60 + action bar 60), and request identity lives in `lblScreenTitle_*` ("… - ID: n - Title"), which costs no extra height. In both, the block is beige `RGBA(241, 239, 232, 1)` — the "request as it stands" surface `RequestDetailScreen` uses — while the invoice-parsing work area (`rowParseInvoice_ISS` / `rowParseInvoice_PE`) is white, so the reference half and the working half read as different surfaces.

Neither block has a flexible child: every child sets `FillPortions: =0` and a real `Height`, so the repeated height sum on `rowRequestInfo_*` **and** `conRequestInfoFields_*` must be ≥ the true total or the last rows clip. Change a row height and all the copies move together.

Where they differ:

| | `InvoiceSubmissionScreen` | `ProcurementExecutionScreen` |
|---|---|---|
| Entered from | `HomeScreen` (3 call sites) and `ProcurementFollowUpScreen`'s invoice gate; `btnBack_ISS` → back to whichever, via `gInvoiceFromPFU` | `RequestDetailScreen` only; `btnBack_PE` → `RequestDetailScreen` |
| Field rows | 4 × 60 = `336` base | 4 × 60 = `336` base, plus Invoice Region (feeds `Submit_Invoice`) |
| Conditional terms | `32` deferred-note, requirement files, `56` remittance | requirement files, `56` requester-invoice link, `56` remittance |
| Requester invoice | already shown by `rowRequesterInvoicePreview_ISS` | `rowReqInvoiceLink_PE` — the only place it appears |

On `ProcurementExecutionScreen` the information was technically reachable (Back does return to the detail screen), but going there mid-form loses the decision radio, six checklist boxes and the supplier summary — `OnVisible` resets all of them. That is the cost the block removes, not a dead end.

### `InvoiceSubmissionScreen` specifics

The field list is: requester, department, category, cost center, project (name resolved via `LookUp(Project_List, …)`), procurement type, invoice type, purchase accordance, estimated cost + currency, budget reference, required delivery date, delivery location, preferred supplier, manager approver (or "Skipped"), submitted-on, the full procurement description, the requirement-file links, and `rowReqRemittance_ISS` (`RemittanceURL` when non-blank — not a requester field, but Procurement needs to know payment already went out before submitting the invoice). Everything else downstream (`InvoiceMode`, approvals, receipt history) is deliberately **not** here — that is what `RequestDetailScreen` is for.

The block replaced `rowInfoBlock_ISS`, pinned at `Y: =60` with a 180/212px height that ate a quarter of the 768px screen before any invoice work was visible.

This mirrors the sibling `procurement-raw-materials` app's `InvoiceSubmissionScreen` control-for-control, apart from that app's raw-material line-items table and the fields it doesn't have (Department, Category). Keep the two in sync.

### Accounting is a shared queue, not an assignment

There is deliberately **no way to assign a request to a specific Accounting person.** `AccountingHandlerID` is written in exactly one place — `AccountingScreen`'s submit, as `gCurrentEmployee` — so it means *"who completed the accounting step"* and stays blank until the request is `Completed`. `RequestDetailScreen` shows it as "Accounting Completed By".

`ProcurementExecutionScreen` and `InvoiceSubmissionScreen` used to carry an "Assign to Accounting Staff *" picker (`ddAccountingHandler_PE` / `ddAccountingHandler_ISS`, both required to submit) that wrote this field plus the log's `HandoverToID`/`HandoverToIDText`. All of it was removed, because the assignment was never real:
- It was overwritten downstream — by `InvoiceSubmissionScreen`, then unconditionally by `AccountingScreen` with whoever actually clicked submit.
- It gated nothing. `HomeScreen`'s Accounting branch filters on **Status**, not on the assignee, so every Accounting user sees every request in the accounting statuses; `AccountingScreen` has no assignee check either.
- It was picked far too early to be meaningful — at Procurement time the request still has the whole receiving loop ahead of it.

Don't reintroduce a picker without also making it load-bearing: gate `AccountingScreen`, filter `HomeScreen` by assignee, and stop `AccountingScreen` from overwriting the field. Related gap, currently accepted: **nothing notifies Accounting** when a request reaches `Pending Accounting` — they discover work by looking at `HomeScreen`.

`Procurement_ExecutionLog.HandoverToID` / `HandoverToIDText` are now written nowhere and read nowhere; they can be deleted from the list whenever convenient.

### The receiving loop's state — two fields on the request, not the log

`Status` alone can't say where inside "Pending Supplier Follow-up" a request is, and a per-row `LookUp` into the execution log is non-delegable and too slow for `HomeScreen`'s gallery. The loop's position therefore lives in two plain request columns: `ReceiptRoundCount` (rounds submitted so far — the next round is `+1`) and `LatestReceiptDecision` (the most recent round's decision, plain **Text** so both option sets fit).

| `LatestReceiptDecision` | Meaning | Screen |
|---|---|---|
| `Requires Supplier Follow-up` | receipt round `ReceiptRoundCount + 1` is open | `SupplierFollowUpScreen` (Requester / `SFU1AssignedToID`) |
| `Accepted with Adjustment` | Credit Note pending | `ProcurementFollowUpScreen` (Procurement/Admin) |

**Call the Step-5 screen the "Procurement close-out", never a round.** *Round* in this app means one **goods receipt**: it increments `ReceiptRoundCount`, writes a `'Procurement Receipt Rounds'` row with a receiver, receipt date, receipt status, decision and photos. `ProcurementFollowUpScreen` does none of that — it writes only `CreditNote`, `Fulfillment` and `Status` plus a Step-5 log row carrying `ExecutedBy`/`ExecutedAt`/`Notes`. It is a **document step**: Procurement collects the supplier's Credit Note and closes the receiving process for the last round, which is why its log row reuses that round's number instead of taking a new one. Two different kinds of work share the single `Pending Supplier Follow-up` status — receiving (the receiver's rounds) and close-out (Procurement's paperwork) — and `LatestReceiptDecision` is what routes between them. The UI says so too: screen title "Procurement Close-out (Round N)", section header "Procurement Close-out — closing Round N", `HomeScreen`'s pill "Procurement Close-out (Credit Note)", and `RequestDetailScreen`'s Step-5 label "Procurement Close-out". The SharePoint choice value `StepName = "Supplier Follow-up — Procurement"` keeps its old text — renaming it means editing the list's choice column, so it stays until that is done deliberately.

**Always gate on the pair `(Status, LatestReceiptDecision)`, never on the decision alone** — `gSFURoundOpen` and `gPFUPending` both do. `LatestReceiptDecision` is *historical*: it records what the latest round decided and is never cleared, so after Procurement uploads the Credit Note it still reads `Accepted with Adjustment` forever. What actually consumes the pending state is `Status` moving on. The receipt-round side is self-clearing for a different reason — the decision is **overwritten** every submit, so `Requires Supplier Follow-up` always describes the newest round.

Both submit buttons re-read the request and abort if it moved under them (`btnSubmitStep1_SFU` checks `ReceiptRoundCount`, `btnSubmit_PFU` checks `Status`/`LatestReceiptDecision`), so two people acting at once can't create a duplicate round or a duplicate close-out.

**The Credit Note step is gated on the Official Invoice.** A Credit Note is an *adjustment to an invoice*, so the invoice has to exist before one can be recorded against it — that, not screen order, is why Procurement must submit the Official Invoice **before** the Credit Note. Note the ordering isn't automatic: a `Deferred` invoice is deliberately submitted *after* goods receipt, and it is goods receipt that decides whether a follow-up round happens at all, so arriving at this screen with no invoice on file is a normal state, not a broken one. Before this gate existed, the close-out simply routed such requests onward to `Pending Invoice`. `OnVisible` computes `gPFUInvoiceMissing = gPFUPending && (Not(InvoiceSubmitted) || IsBlank(OfficialInvoiceLink))` — the constraint is that the invoice **data** exists, and the two columns are written together everywhere (`ProcurementExecutionScreen` paths A/C, `InvoiceSubmissionScreen`) but only the link proves it, so both are checked. While it's true the screen swaps the Remarks + Credit Note form and `btnSubmit_PFU` for the amber `rowInvoiceRequired_PFU` block, whose "Submit Invoice →" button does `Set(gInvoiceFromPFU, true); Navigate(InvoiceSubmissionScreen)`. `btnSubmit_PFU` re-checks the same pair on `wLatest` alongside the `(Status, LatestReceiptDecision)` guard, since both can go stale while the screen is open. Because the invoice is guaranteed by then, the close-out patches `Status = "Pending Accounting"` unconditionally — the old `If(!InvoiceSubmitted, "Pending Invoice", …)` branch is unreachable here (`GoodsReceiptScreen` and `SupplierFollowUpScreen` still have it; their "Accepted" path has no such gate).

`gInvoiceFromPFU` is the return ticket for that detour: `InvoiceSubmissionScreen`'s `btnBack_ISS` and all three submit-path navigations do `If(gInvoiceFromPFU, Refresh(Procurement_Requests); Navigate(ProcurementFollowUpScreen), Navigate(HomeScreen))` — the `Refresh` is what makes `ProcurementFollowUpScreen.OnVisible`'s `LookUp` see `InvoiceSubmitted = true` and unlock the form. It is reset to `false` in `App.OnStart`, in `HomeScreen.OnVisible` (which covers all three of that screen's entries into `InvoiceSubmissionScreen`) and in `ProcurementFollowUpScreen.OnVisible`. This is the one place `InvoiceSubmissionScreen` is entered from somewhere other than `HomeScreen`, and the one divergence from the sibling `procurement-raw-materials` app's copy of that screen.

Each round writes two rows, with a strict division of labour: the **`'Procurement Receipt Rounds'` row owns everything about receiving** — receiver, date, status, decision, remarks and the **receipt photos** (the attachment form binds to that list) — while the `Procurement_ExecutionLog` row keeps only the generic step trail (`ExecutedBy`, `ExecutedAt`, `Notes`). The round row is therefore `Patch`ed **first**, so the attachment form has a record to submit against.

Don't move the round header onto the log "to match the raw-materials app": there the two lists carry *different grains* (log = one row per round header, rounds list = one row per line item × round), so nothing repeats. This app has no line items, so a round is a single row — put the header on the log as well and every submit writes it twice.

**Executive-payment sub-flow** (`isExecutivePayment` Yes/No field on `Procurement_Requests`): when Executive approves a request that is **both** `ProcurementType = "Invoice Supplied"` **and** over-threshold (`Currency <> "AUD" || EstimatedCost > 10000` — non-AUD currency always qualifies, no FX conversion; AUD only qualifies above 10,000), the request does **not** advance to "Pending Procurement" — it stays `Status = "Pending Executive"` with `isExecutivePayment = true`, and `RequestDetailScreen`/`HomeScreen` display it as **"Pending Payment From Executive"** purely as a computed label (same pattern as the "Supplier Follow-up (Step 1/2)" sub-status — the real `Status` value never changes). Requests with `ProcurementType = "To be sourced by Procurement"` never enter this sub-flow regardless of amount/currency — they always proceed straight to `Pending Procurement`, since Procurement still has to source the item before any payment is relevant. On submitting the decision, `ExecutiveApprovalScreen` also calls `Procurement_Notify_Receipt_Assignee.Run(..., "ExecutivePayment", ...)` to email the approving Executive before navigating to `ExecutivePaymentScreen`. The Executive then uses `ExecutivePaymentScreen` (also reachable via `RequestDetailScreen`'s "Process Payment →" button, shown only when `isExecutivePayment = true`) to upload a remittance advice document; submitting patches `RemittanceURL` and moves `Status` to `"Pending Procurement"` (keeping `isExecutivePayment = true` for history), then calls `Procurement_Notify_Receipt_Assignee.Run(..., "ProcurementExecution", ...)` to notify the Procurement team. `ManagerReviewScreen`'s "Approved (within budget)" fast-track also re-checks the AUD/10,000 threshold (but not `ProcurementType`) — a within-budget approval on an over-threshold request still routes to `Pending Executive` instead of skipping straight to `Pending Procurement`, so the payment step can't be bypassed for requests that do end up qualifying at the Executive step.

The `formExecutivePayment.OnSuccess` patch on `ExecutivePaymentScreen` must re-fetch the record with `LookUp(Procurement_Requests, ID = gSelectedRequest.ID)` rather than reusing `gSelectedRequest` as the Patch base — `SubmitForm(formExecutivePayment)` already wrote the `Attachments` field earlier in the same handler chain, which bumps the item's SharePoint version, so patching against the pre-submit `gSelectedRequest` throws "Conflicts exist with changes on the server, please reload".

`RemittanceURL` is shared between two independent producers: `ExecutivePaymentScreen` (this sub-flow) and `ProcurementExecutionScreen`'s own "Remittance Advice Document" upload (Path C / `locIsViaRequester`, when Procurement proceeds with a requester-supplied invoice). When `isExecutivePayment = true`, `ProcurementExecutionScreen` hides its own remittance upload requirement entirely (Executive's upload already satisfies it) and reuses the existing `RemittanceURL` instead of asking Procurement to attach a second document — see `rowFormRemittance_PE` / `rowExecutiveRemittanceInfo_PE` and the `wURL` branch in `formRemittance.OnSuccess`. `RequestDetailScreen` shows both `RequesterInvoiceURL` (`rowInvoiceLink`) and `RemittanceURL` (`rowRemittanceLink`) as clickable links whenever each field is non-blank — visibility does **not** depend on `InvoiceMode` (which is only set later, in `ProcurementExecutionScreen`, so gating on it would hide the Requester's invoice from Manager/Executive during review).

**Limited-use goods** (`UsageExpiryDate` Date field on `Procurement_Requests`): some goods are only usable up to a fixed date — event materials, for instance — and warehouse staff must not put them into storage afterwards. Procurement classifies this on `ProcurementExecutionScreen` (`rowLimitedUse_PE`): a `rdoLimitedUse_PE` Yes/No radio group defaulting to `"No"`, sharing one horizontal row (`rowLimitedUseInputs_PE`) with the `dpLimitedUseDate_PE` date picker that appears — and becomes required — when set to `"Yes"`. The value is written by **both** submit paths — `formConfirmation.OnSuccess` (Path A/B) and `formRemittance.OnSuccess` (Path C) — as `If(rdoLimitedUse_PE.Selected.Value = "Yes", dpLimitedUseDate_PE.SelectedDate, Blank())`, so a change to one Patch must be mirrored in the other. There is no separate Yes/No column: **a non-blank `UsageExpiryDate` is the flag**. Downstream it's read-only — `HomeScreen`'s `lblLimitedUseBadge` (red badge in `rowTitleLine`) and `RequestDetailScreen`'s `rowLimitedUse_RD`, both gated on `Not(IsBlank(...UsageExpiryDate))`; `rowLimitedUse_RD`'s 60px must stay in the `rowInfoBlock.LayoutMinHeight` sum.

Routing relies on these status strings being exact and consistent across `HomeScreen` (filters + gallery `Items` per-role filter), each action screen, and the `Switch`/`If` color maps. **When changing status names or the flow, update every screen that references the string** — there is no shared constant.

## Role-based visibility (HomeScreen)

The gallery `Items` filters `Procurement_Requests` differently per `gUserRole`:
- **Manager** → requests where `ManagerApproverID.Id = gCurrentEmployee.ID`.
- **Procurement / Accounting** → requests in their relevant statuses onward.
- **Executive / Admin** → all requests.
- **Requester (default)** → own requests (`RequesterEmail = gCurrentEmployee.Email`). Non-special-role employees who are assigned as a goods receipt / supplier follow-up receiver also see those requests via `GRAssignedToID.Id = gCurrentEmployee.ID || SFU1AssignedToID.Id = gCurrentEmployee.ID`.

Filter buttons and "+ New Request" are shown/hidden by role. Keep the per-role `Items` filter and the filter-button `Visible` rules in sync.

## Conventions

- All UI text, field names, comments, and code are **English** (per global instruction). Vietnamese appears only as SharePoint system-column internal names and the `vi-VN` UserLocale.
- Patches use the `With({wPatched: Patch(...)}, If(IsBlank(wPatched.ID), Notify(error), ...success...))` pattern to guard against write failures — follow it for new writes.
- `RequestIDText` (text copy of the request ID) is used to look up log rows: `LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = N)`.
- Colors are inline `RGBA(...)`; brand purple is `RGBA(83, 74, 183, 1)`.
