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

Full column-level schema (types, choice values, join keys): `docs/sharepoint-schema.md`.

**Power Automate** flows called from Power Fx:

Invoice flows (called from `ProcurementExecutionScreen`):
- `Parse_Invoice.Run(invoiceUrl, requestId)` — AI invoice extraction.
- `Submit_Invoice.Run(...)` — writes parsed invoice data.

Assignment notification flow (called from `GoodsReceiptScreen` and `SupplierFollowUpScreen`):
- `Procurement_Notify_Receipt_Assignee.Run(assigneeEmail, assigneeName, requestTitle, requestId, notificationType, deliveryDate, category)`
  - `notificationType = "GoodsReceipt"` — notify new assignee they must perform Goods Receipt & Acceptance.
  - `notificationType = "SupplierFollowUp"` — notify new assignee they must perform Supplier Follow-up Round 2.
  - `notificationType = "Unassigned"` — notify previous assignee they no longer need to act (email only, no Adaptive Card).
  - Called on `btnSaveAssignment_GR` / `btnSaveAssignment_SFU` (assign new person) and `btnIWillReceive_GR` / `btnIWillReceive_SFU` (requester takes back the task).
  - Flow sends Outlook email + Teams Adaptive Card for assignment types; email only for Unassigned.
  - Connection: `app.admin@maxbiocare.com` pinned in "Run only users" — not invoker-provided.

**Important:** SharePoint internal field names are Vietnamese for system columns — `'Tiêu đề'` = Title, `'Tệp đính kèm'` = Attachments. Custom columns are English (`Status`, `EstimatedCost`, `ManagerApproverID`, etc.). Reference Vietnamese-named columns with single quotes in Power Fx.

## Global state (set in App.OnStart)

- `gCurrentEmployee` — row from `Employee List` matched by `User().Email`.
- `gCurrentUser` / `gUserRole` — row from `Procurement_User`; defaults to role `"Requester"` if not found.
- `gIsSpecialRole` — true for Manager/Executive/Procurement/Accounting/Admin (drives toolbar/filter visibility).
- `gSelectedRequest` — the request being viewed/acted on (set before `Navigate`).
- `gStatusFilter`, `gParsingInvoice`, `gHasInvoiceResult`, `gInvoiceResult`, `gShowRejectReason`.

If `gCurrentEmployee.ID` is blank, the user sees an "account not found" message and no UI — the app is membership-gated by the `Employee List`.

## The workflow — this is the core domain model

Requests move through a `Status` choice field. Each screen patches `Procurement_Requests.Status` and writes a log row. The status string is also the value of the `HomeScreen` filter buttons and the gallery color coding.

```
RequestFormScreen (Requester submits)
   │  Auto-skip to Executive if: PurchaseAccordance = "Urgent"/"Unplanned",
   │  OR (EstimatedCost > 5000 AND Currency = "AUD")  → sets SkippedManagerReview
   ▼
Pending Manager ──(ManagerReviewScreen)──┐
   │ "Approved (within budget)" → Pending Procurement
   │ otherwise                   → Pending Executive
   ▼
Pending Executive ──(ExecutiveApprovalScreen)──┐
   │ Reject → Rejected
   │ Approve / Approve with conditions → Pending Procurement (ConditionsText)
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
