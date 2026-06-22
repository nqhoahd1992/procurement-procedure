# SharePoint Lists — Schema Reference

Data model for the Max Biocare Procurement Procedure app. Site: `maxbiocare.sharepoint.com/sites/Powerapps`.

> **Source of truth is SharePoint itself.** This document is reconstructed from how columns are read/written in `Src/*.pa.yaml`, so it covers the fields the app actually uses. Column *types* are inferred from usage; choice option sets marked "(full set in SharePoint)" list only the values referenced in code. Keep this file in sync when you add/rename columns in SharePoint.

## Conventions

- **System columns use Vietnamese internal names** — quote them in Power Fx: `'Tiêu đề'` (Title), `'Tệp đính kèm'` (Attachments).
- **Person/Lookup** columns are written as `{Id: ..., Value: ...}` and read as `Col.Value` / `Col.Id`.
- **Choice** columns are written as `{Value: "..."}` and read as `Col.Value`.
- `RequestIDText` (a plain-text copy of the request ID) is the join key used to look up log rows, because text comparison delegates reliably: `LookUp(Procurement_ExecutionLog, RequestIDText = Text(request.ID) && StepNumber = N)`.

---

## `Procurement_Requests` — central request record

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | Auto-built: `<employee> - <Category> - <dd/mm/yyyy>` |
| `RequesterID` | Person | `{Id, Value}` — links to Employee List |
| `RequesterEmail` | Text | `User().Email` of submitter; used for "my requests" filter |
| `Department` | Text | Plain text from dropdown |
| `ProcurementDescription` | Text (multiline) | |
| `BudgetReference` | Text | |
| `Category` | Choice | (full set in SharePoint) |
| `PurchaseAccordance` | Choice | Includes `Urgent`, `Unplanned` (these auto-escalate to Executive) |
| `ProcurementType` | Choice | Includes `Invoice Supplied` |
| `InvoiceType` | Choice | Includes `Official Invoice`; blank unless ProcurementType = Invoice Supplied |
| `InvoiceRegion` | Choice | Invoice target market |
| `DeliveryLocation` | Choice | |
| `Currency` | Choice | Includes `AUD` |
| `EstimatedCost` | Number | |
| `RequiredDeliveryDate` | Date | |
| `PreferredSupplier` | Text | Supplier title |
| `ManagerApproverID` | Person | `{Id, Value}`; blank when manager review is skipped |
| `SkippedManagerReview` | Yes/No | True when Urgent/Unplanned **or** (EstimatedCost > 5000 AND Currency = AUD) |
| `Status` | Choice | **Drives the whole workflow** — see value list below |
| `ConditionsText` | Text | Set when Executive picks "Approve with conditions" |
| `OfficialInvoiceLink` | Text (URL) | SharePoint link to the official invoice |
| `AccountingHandlerID` | Person | `{Id, Value}` — chosen by Procurement |
| `ProcurementExecutedBy` | Person | `{Id, Value}` |
| `ProcurementExecutedAt` | DateTime | |
| `AccountingCompletedAt` | DateTime | |
| `GoodsReceiptBy` | Text | |
| `GoodsReceiptDate` | Date | |
| `GoodsReceiptStatus` | Choice | |
| `GoodsAcceptanceDecision` | Choice | `Accepted`, `Rejected`, `Requires Supplier Follow-up` |
| `GoodsReceiptRemarks` | Text | |
| `GoodsReceiptAt` | DateTime | |
| `FollowUpReceiptBy` | Text | |
| `FollowUpReceiptDate` | Date | |
| `FollowUpAcceptanceDecision` | Choice | `Accepted`, `Accepted with Adjustment` |
| `Fulfillment` | Choice | `Fulfilled`, `Fulfilled with Adjustment` |
| `FollowUpRemarks` | Text | |
| `FollowUpReceiptAt` | DateTime | |
| `SupplierFollowUpNotes` | Text | |
| `CreditNote` | Text | Required when follow-up decision = adjustment |
| `FollowUpCompletedAt` | DateTime | |
| `PurchaseRequestLink` | Text (URL) | |
| `Tệp đính kèm` (Attachments) | Attachments | Invoice/supporting files; written via `Form1` + `SubmitForm` (Patch cannot write attachments) |

### `Status` choice values (exact strings — used as literals across all screens)

`Pending Manager` → `Pending Executive` → `Pending Procurement` → `Pending Accounting` → `Goods Receipt & Acceptance` → `Pending Supplier Follow-up` → `Completed`, plus terminal `Rejected`.

> Changing any of these strings means updating every screen's filters, color maps, and `Patch`/`If`/`Switch` logic — there is no shared constant.

---

## `Procurement_User` — role assignment

| Column | Type | Notes |
|---|---|---|
| `EmployeeID` | Lookup | `{Id, Value}` → Employee List |
| `Role` | Choice | `Requester`, `Manager`, `Executive`, `Procurement`, `Accounting`, `Admin` |
| `IsActive` | Yes/No | Manager dropdown filters on `Role.Value = "Manager" && IsActive` |

Resolved in `App.OnStart` → `gCurrentUser` / `gUserRole`. Employees not found here default to role `Requester`.

---

## `Procurement_ApprovalLog` — manager & executive decisions

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | `Step <n> - <decision> - <request title>` |
| `RequestID` | Lookup | `{Id, Value}` |
| `RequestIDText` | Text | Join key |
| `StepNumber` | Number | `2` = Manager, `3` = Executive |
| `ApproverID` | Person | `{Id, Value}` |
| `Decision` | Choice | Manager: `Approved (within budget)`, … · Executive: `Reject`, `Approve with conditions`, … |
| `ManagerRemarks` | Text | Step 2 only |
| `ApprovalConditions` | Text | Step 3 only (when "Approve with conditions") |
| `RejectionReason` | Text | Step 3 only (when "Reject") |

---

## `Procurement_ExecutionLog` — procurement / receipt / follow-up steps

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | `Step <n> - <name> - <request title>` |
| `RequestID` | Lookup | `{Id, Value}` |
| `RequestIDText` | Text | Join key |
| `StepNumber` | Number | `1` Procurement Execution · `2` Accounting Handover · `3` Goods Receipt · `4` Supplier Follow-up (Requester) · `5` Supplier Follow-up (Procurement) |
| `StepName` | Choice | Matches the step |
| `ExecutedBy` | Person | `{Id, Value}` |
| `ExecutedAt` | DateTime | |
| `Notes` | Text | |
| `HandoverToID` | Person | Step 1 — accounting handler `{Id, Value}` |
| `HandoverToIDText` | Text | Step 1 |
| `SupplierSummary` | Text | Step 1 |
| `PurchaseOrderLink` | Text (URL) | Step 1 |

The presence of step-4 / step-5 rows is what distinguishes the two stages of the supplier follow-up flow (the app `LookUp`s these by `StepNumber`).

---

## `Procurement_InvoiceData` — extracted invoice fields

**Not written directly by the app.** It is populated by the **`Submit_Invoice`** Power Automate flow, called from `ProcurementExecutionScreen` with these arguments (in order): invoice link, request ID, suggested filename, invoice date, supplier name, billed-to, invoice number, total amount, tax amount, currency, AI `jobId`, AI `confidenceScore`, attention, (reserved ""), invoice region, ABN.

The **`Parse_Invoice`** flow returns the extraction result into `gInvoiceResult` (fields used: `.jobId`, `.confidenceScore`).

---

## `Suppliers`

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | Supplier name; bound to the "Preferred Supplier" dropdown |

## `Employee List` (standard SharePoint people directory)

| Column | Type | Notes |
|---|---|---|
| `ID` | Number | Matched against `Procurement_User.EmployeeID` |
| `Tiêu đề` (Title) | Text | Employee display name |
| `Email` | Text | Matched against `User().Email` in `App.OnStart` |
