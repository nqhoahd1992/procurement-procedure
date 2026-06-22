# SharePoint Lists & Flows — Schema Reference

Authoritative data model for the Max Biocare Procurement Procedure app, reconstructed from the unpacked `References/DataSources.json`. Site: `maxbiocare.sharepoint.com/sites/Powerapps`.

> Only **custom / app-relevant columns** are listed. Every SharePoint list also carries the standard system columns (`ID`, `Created`, `Modified`, `Author`, `Editor`, `Tệp đính kèm`/Attachments, content-type, moderation, etc.) — omitted here for clarity. Update this file whenever a SharePoint column is added/renamed.

## Conventions

- **System columns use Vietnamese internal names** — quote them in Power Fx: `'Tiêu đề'` (Title), `'Tệp đính kèm'` (Attachments).
- **Choice** columns: write `{Value: "..."}`, read `Col.Value`.
- **Lookup / Person** columns: write `{Id: ..., Value: ...}`, read `Col.Value` / `Col.Id`. (These are SharePoint *lookup* columns pointing at another list — `Col.Value` resolves to the target's Title or ID per the `OdataQueryName` noted below.)
- **Required** columns are flagged ⚠ — a `Patch` that omits them fails.
- `RequestIDText` (plain-text copy of the request ID) is the delegable join key for log lookups: `LookUp(Procurement_ExecutionLog, RequestIDText = Text(request.ID) && StepNumber = N)`.

---

## `Procurement_Requests` — central request record

Required ⚠: `Status`, `RequesterEmail`, `ProcurementType`, `ProcurementDescription`, `Category`, `PurchaseAccordance`, `EstimatedCost`, `Currency`, `RequiredDeliveryDate`, `DeliveryLocation`, `RequesterID`, `InvoiceRegion`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | Auto-built: `<employee> - <Category> - <dd/mm/yyyy>` |
| `Status` ⚠ | Choice | **Drives the workflow** — value list below |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `Department` ⚠* | Text | Plain text (\*not required in schema, but set on submit) |
| `ProcurementType` ⚠ | Choice | incl. `Invoice Supplied` |
| `InvoiceType` | Choice | incl. `Official Invoice`; blank unless ProcurementType = Invoice Supplied |
| `InvoiceLink` | Text (URL) | Original/uploaded invoice link |
| `OfficialInvoiceLink` | Text (URL) | Final official invoice link set by Procurement |
| `ProcurementDescription` ⚠ | Text (multiline) | |
| `Category` ⚠ | Choice | |
| `PreferredSupplier` | Text | Supplier title |
| `PurchaseAccordance` ⚠ | Choice | incl. `Urgent`, `Unplanned` (auto-escalate to Executive) |
| `EstimatedCost` ⚠ | Number | |
| `Currency` ⚠ | Choice | incl. `AUD` |
| `BudgetReference` | Text | |
| `RequiredDeliveryDate` ⚠ | Date | |
| `DeliveryLocation` ⚠ | Choice | |
| `RequesterID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `ManagerApproverID` | Lookup→Employee List | blank when manager review skipped |
| `SkippedManagerReview` | Yes/No | true when Urgent/Unplanned **or** (EstimatedCost > 5000 AND Currency = AUD) |
| `ProcurementExecutedBy` | Lookup→Employee List | |
| `ProcurementExecutedAt` | DateTime | |
| `AccountingHandlerID` | Lookup→Employee List | chosen by Procurement |
| `AccountingCompletedAt` | DateTime | |
| `ConditionsText` | Text (multiline) | set on "Approve with conditions" |
| `RequestID` | Lookup (→ID) | present in schema; not used by the app flow |
| `PurchaseRequestLink` | Text (URL) | |
| `GoodsReceiptBy` | Text | |
| `GoodsReceiptDate` | DateTime | |
| `GoodsReceiptStatus` | Choice | |
| `GoodsAcceptanceDecision` | Choice | `Accepted`, `Rejected`, `Requires Supplier Follow-up` |
| `GoodsReceiptRemarks` | Text (multiline) | |
| `GoodsReceiptAt` | DateTime | |
| `FollowUpReceiptBy` | Text | |
| `FollowUpReceiptDate` | DateTime | |
| `FollowUpAcceptanceDecision` | Choice | `Accepted`, `Accepted with Adjustment` |
| `CreditNote` | Text | required when follow-up decision = adjustment |
| `Fulfillment` | Choice | `Fulfilled`, `Fulfilled with Adjustment` |
| `FollowUpRemarks` | Text (multiline) | |
| `FollowUpReceiptAt` | DateTime | |
| `SupplierFollowUpNotes` | Text (multiline) | |
| `FollowUpCompletedAt` | DateTime | |
| `InvoiceRegion` ⚠ | Choice | invoice target market |
| `Tệp đính kèm` (Attachments) | Attachments | invoice/supporting files; written via `Form1`+`SubmitForm` (Patch can't write attachments) |

### `Status` choice values (exact strings — used as literals across all screens)

`Pending Manager` → `Pending Executive` → `Pending Procurement` → `Pending Accounting` → `Goods Receipt & Acceptance` → `Pending Supplier Follow-up` → `Completed`, plus terminal `Rejected`.

> Changing any string means updating every screen's filters, color maps, and `Patch`/`If`/`Switch` logic — there is no shared constant.

---

## `Procurement_User` — role assignment

Required ⚠: `Role`, `IsActive`, `EmployeeID`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | |
| `Role` ⚠ | Choice | `Requester`, `Manager`, `Executive`, `Procurement`, `Accounting`, `Admin` |
| `IsActive` ⚠ | Yes/No | default true; manager picker filters `Role.Value="Manager" && IsActive` |
| `EmployeeID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `Note` | Text | |

Resolved in `App.OnStart` → `gCurrentUser` / `gUserRole`. Employees absent here default to role `Requester`.

---

## `Procurement_ApprovalLog` — manager & executive decisions

Required ⚠: `RequestID`, `StepNumber`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | `Step <n> - <decision> - <request title>` |
| `RequestID` ⚠ | Lookup (→Title) | `{Id,Value}` |
| `StepNumber` ⚠ | Number | **`2` = Manager, `3` = Executive** (documented in the column itself) |
| `ApproverID` | Lookup→Employee List | |
| `Decision` | Choice | Manager: `Approved (within budget)`, … · Executive: `Reject`, `Approve with conditions`, `Approve` |
| `ApprovalConditions` | Text (multiline) | step 3, "Approve with conditions" |
| `RejectionReason` | Text (multiline) | step 3, "Reject" |
| `ManagerRemarks` | Text (multiline) | step 2 |
| `RequestIDText` | Text | join key |

---

## `Procurement_ExecutionLog` — procurement / receipt / follow-up steps

Required ⚠: none.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | `Step <n> - <name> - <request title>` |
| `RequestID` | Lookup (→ID) | `{Id,Value}` |
| `RequestIDText` | Text | join key |
| `StepNumber` | Number | `1` Procurement Execution · `2` Accounting Handover · `3` Goods Receipt · `4` Supplier Follow-up (Requester) · `5` Supplier Follow-up (Procurement) |
| `StepName` | Choice | matches the step |
| `ExecutedBy` | Lookup→Employee List | |
| `ExecutedAt` | DateTime | |
| `HandoverToID` | Lookup→Employee List | step 1 — accounting handler |
| `HandoverToIDText` | Text | step 1 |
| `SupplierSummary` | Text (multiline) | step 1 |
| `PurchaseOrderLink` | Text (URL) | step 1 |
| `Notes` | Text (multiline) | |

The presence of step-4 / step-5 rows distinguishes the two stages of the supplier follow-up flow (`LookUp` by `StepNumber`).

---

## `Procurement_InvoiceData` — extracted invoice fields

Required ⚠: none. **Not written by app `Patch`** — populated by the `Submit_Invoice` flow (see below). Column names differ from the app's variable names — note `VendorName` and `GSTAmount`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | |
| `RequestID` | Lookup (→Title) | |
| `RequestIDText` | Text | join key |
| `InvoiceNumber` | Text | |
| `InvoiceDate` | Date | |
| `VendorName` | Text | supplier name (flow arg `supplierName`) |
| `BilledTo` | Text | |
| `Attention` | Text | |
| `Description` | Text (multiline) | |
| `TotalAmount` | Number | |
| `GSTAmount` | Number | tax amount (flow arg `taxAmount`) |
| `Currency` | Text | |
| `ParsedAt` | Date | |
| `InvoiceLink` | Text (URL) | |
| `ABN` | Text | supplier ABN |

---

## `Employee List` — staff directory

Required ⚠: none. **Internal names `email` and `department` are lowercase** (display names are capitalized).

| Column (internal) | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | employee display name |
| `email` | Text | matched against `User().Email` in `App.OnStart` |
| `department` | Choice | |
| `JobTitle` | Text | |
| `City` | Text | |
| `Country` | Text | |
| `EmployeeCode` | Text | |

---

## `Suppliers`

Required ⚠: none. Bound to the "Preferred Supplier" picker via `Sort(Suppliers, 'Tiêu đề')`.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | supplier name |
| `Code` | Text | |
| `AccountName` | Text | |
| `TaxId` | Text | |
| `ABN` | Text | |
| `Country` | Choice | |
| `PicName` | Text | person in charge |
| `Email` | Text | |
| `Phone` | Text | |
| `BillingAddress` / `DeliveryAddress` | Text | |
| `ContractType` / `ScopeOfWork` | Text | |
| `Industry` / `BusinessType` | Text | |
| `PaymentTerms` / `Margin` | Text | |
| `CompanyRegNo` | Text | |
| `MisaCrmLink` | Text (URI) | |
| `Note` | Text (multiline) | |

---

## Power Automate flows (Logic flows connector)

Both are called from `ProcurementExecutionScreen` via `<Flow>.Run(...)`.

### `Parse_Invoice.Run(invoiceLink, requestId)` — AI invoice extraction

| Input | Type | |
|---|---|---|
| `text` | string | invoice link |
| `number` | number | request ID |

**Returns** (`gInvoiceResult`): `invoiceNumber`, `invoiceDate`, `totalAmount` (n), `taxAmount` (n), `supplierName`, `supplierABN`, `billedTo`, `currency`, `confidenceScore` (n), `jobId`, `attention`, `suggestedFilename`.

### `Submit_Invoice.Run(...)` — writes the official invoice + `Procurement_InvoiceData`

16 positional args (the trigger names them `text`, `number`, `text_1`…); the app passes them in this order:

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
| 14 | `text_9` | (reserved — empty string) |
| 15 | `text_10` | invoice region |
| 16 | `text_11` | ABN |

**Returns**: `newinvoicelink` (string).
