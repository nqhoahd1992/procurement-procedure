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
- `gStatusFilter`, `gParsingInvoice`, `gHasInvoiceResult`, `gInvoiceResult`, `gShowRejectReason`.
- `gPendingAttachments` / `gPendingRequestFiles` / `gPendingInvoiceName` — `RequestFormScreen`'s two attachment controls and the bookkeeping that lets them share one SharePoint column. The screen has **two** cards bound to `Procurement_Requests.{Attachments}`, in two forms: `Form1`/`DataCardValue15` (the invoice, `MaxAttachments: =1`, only visible and required when `ProcurementType = "Invoice Supplied"`) and `formRequestFiles`/`attRequestFiles` (**Requirement Files** — optional, `MaxAttachments: =10`, always visible). Both keep `Default: =ThisItem.Attachments` with `Items: =If(IsBlank(Parent.Default), gPending…, Parent.Default)`: the request doesn't exist while the form is being filled, so each control is in `NewForm` mode, and the `EditForm`+`SubmitForm` that runs after the request row is created would otherwise wipe the user's picks — the `gPending…` var carries them across that reset. The request row itself is created by `Patch`, not by either form; both forms are pure attachment vehicles.
  - **Submit is chained, never parallel.** `btnSubmit_1` submits `formRequestFiles`, and *its* `OnSuccess` runs `EditForm(Form1); SubmitForm(Form1)`; `Form1.OnSuccess` is the terminal step (patches `RequesterInvoiceURL`, resets every input, navigates). Two `SubmitForm`s fired at once against the same item race each other. When there are no requirement files the chain is skipped and `Form1` is submitted directly — `Form1` is always submitted, even on the non-invoice path where it sits inside a hidden container, because its `OnSuccess` is what navigates home.
  - `gPendingInvoiceName` exists because `Form1` now submits **after** the requirement files are already on the item. The old `First(Form1.LastSubmit.Attachments).AbsoluteUri` no longer picks the invoice — and per `docs/powerapps-form-attachment-pattern.md` rule 3 it was never safe anyway, since SharePoint returns `LastSubmit.Attachments` in **alphabetical** order. The name is captured from `Last(DataCardValue15.Attachments).Name` **before** submit and the URL composed the way every other upload does: `gSharePointAttachmentBase & Text(ID) & "/" & EncodeUrl(name)`.
- `colRequirementFiles` — `RequestDetailScreen`'s "Requirement Files (Requester)" list (`rowRequirementFiles_RD` / `galRequirementFiles_RD`, just above `rowInvoiceLink`). `{FileName, FileURL}`, built in `OnVisible` by splitting `gSelectedRequest.RequirementFiles` on `|` and composing each URL from `gSharePointAttachmentBase`; seeded with a literal in `App.OnStart` for its design-time schema. The row's `Visible`/`Height` and the `CountRows(colRequirementFiles)` term in `rowInfoBlock.LayoutMinHeight` must stay in sync — that container sizes itself by summing its visible children.
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
Pending Supplier Follow-up ──(SupplierFollowUpScreen, 2 steps)──┐
   │ Step 1 = Requester detail (ExecutionLog StepNumber 4)
   │ Step 2 = Procurement      (ExecutionLog StepNumber 5)
   │ Accepted → Completed; else stays Pending Supplier Follow-up
   ▼
Completed
```

**Executive-payment sub-flow** (`isExecutivePayment` Yes/No field on `Procurement_Requests`): when Executive approves a request that is **both** `ProcurementType = "Invoice Supplied"` **and** over-threshold (`Currency <> "AUD" || EstimatedCost > 10000` — non-AUD currency always qualifies, no FX conversion; AUD only qualifies above 10,000), the request does **not** advance to "Pending Procurement" — it stays `Status = "Pending Executive"` with `isExecutivePayment = true`, and `RequestDetailScreen`/`HomeScreen` display it as **"Pending Payment From Executive"** purely as a computed label (same pattern as the "Supplier Follow-up (Step 1/2)" sub-status — the real `Status` value never changes). Requests with `ProcurementType = "To be sourced by Procurement"` never enter this sub-flow regardless of amount/currency — they always proceed straight to `Pending Procurement`, since Procurement still has to source the item before any payment is relevant. On submitting the decision, `ExecutiveApprovalScreen` also calls `Procurement_Notify_Receipt_Assignee.Run(..., "ExecutivePayment", ...)` to email the approving Executive before navigating to `ExecutivePaymentScreen`. The Executive then uses `ExecutivePaymentScreen` (also reachable via `RequestDetailScreen`'s "Process Payment →" button, shown only when `isExecutivePayment = true`) to upload a remittance advice document; submitting patches `RemittanceURL` and moves `Status` to `"Pending Procurement"` (keeping `isExecutivePayment = true` for history), then calls `Procurement_Notify_Receipt_Assignee.Run(..., "ProcurementExecution", ...)` to notify the Procurement team. `ManagerReviewScreen`'s "Approved (within budget)" fast-track also re-checks the AUD/10,000 threshold (but not `ProcurementType`) — a within-budget approval on an over-threshold request still routes to `Pending Executive` instead of skipping straight to `Pending Procurement`, so the payment step can't be bypassed for requests that do end up qualifying at the Executive step.

The `formExecutivePayment.OnSuccess` patch on `ExecutivePaymentScreen` must re-fetch the record with `LookUp(Procurement_Requests, ID = gSelectedRequest.ID)` rather than reusing `gSelectedRequest` as the Patch base — `SubmitForm(formExecutivePayment)` already wrote the `Attachments` field earlier in the same handler chain, which bumps the item's SharePoint version, so patching against the pre-submit `gSelectedRequest` throws "Conflicts exist with changes on the server, please reload".

`RemittanceURL` is shared between two independent producers: `ExecutivePaymentScreen` (this sub-flow) and `ProcurementExecutionScreen`'s own "Remittance Advice Document" upload (Path C / `locIsViaRequester`, when Procurement proceeds with a requester-supplied invoice). When `isExecutivePayment = true`, `ProcurementExecutionScreen` hides its own remittance upload requirement entirely (Executive's upload already satisfies it) and reuses the existing `RemittanceURL` instead of asking Procurement to attach a second document — see `rowFormRemittance_PE` / `rowExecutiveRemittanceInfo_PE` and the `wURL` branch in `formRemittance.OnSuccess`. `RequestDetailScreen` shows both `RequesterInvoiceURL` (`rowInvoiceLink`) and `RemittanceURL` (`rowRemittanceLink`) as clickable links whenever each field is non-blank — visibility does **not** depend on `InvoiceMode` (which is only set later, in `ProcurementExecutionScreen`, so gating on it would hide the Requester's invoice from Manager/Executive during review).

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
