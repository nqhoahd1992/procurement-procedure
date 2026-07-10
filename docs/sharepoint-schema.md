# SharePoint Lists & Flows — Schema Reference

Authoritative data model for the Max Biocare Procurement Procedure app, reconstructed from the unpacked `References/DataSources.json`. Site: `maxbiocare.sharepoint.com/sites/Powerapps`.

> Only **custom / app-relevant columns** are listed. Every SharePoint list also carries the standard system columns (`ID`, `Created`, `Modified`, `Author`, `Editor`, `Attachments`, content-type, moderation, etc.) — omitted here for clarity. Update this file whenever a SharePoint column is added/renamed.

## Conventions

- **System columns use English internal names** (site locale = en) — use `Title` and `Attachments` directly in Power Fx, no quoting needed.
- **Choice** columns: write `{Value: "..."}`, read `Col.Value`.
- **Lookup / Person** columns: write `{Id: ..., Value: ...}`, read `Col.Value` / `Col.Id`. (These are SharePoint *lookup* columns pointing at another list — `Col.Value` resolves to the target's Title or ID per the `OdataQueryName` noted below.)
- **Required** columns are flagged ⚠ — a `Patch` that omits them fails.
- `RequestIDText` (plain-text copy of the request ID) is the delegable join key for log lookups: `LookUp(Procurement_ExecutionLog, RequestIDText = Text(request.ID) && StepNumber = N)`.

---

## `Procurement_Requests` — central request record

Required ⚠: `Status`, `RequesterEmail`, `ProcurementType`, `ProcurementDescription`, `Category`, `PurchaseAccordance`, `EstimatedCost`, `Currency`, `RequiredDeliveryDate`, `DeliveryLocation`, `RequesterID`, `InvoiceRegion`, `CostCenter`, `ProjectID`.

**Not yet created on SharePoint** — `CostCenter` (Text) must be added to this list before `RequestFormScreen`'s `txtCostCenter_1` field can be wired (see row below). Deliberately **Text, not Choice** — same reasoning as `project-list`'s own `Project_List.CostCenter` (also Text, see that repo's schema doc): the value is just passed through, so there's no Choice value set to keep in sync across the two separate Power Apps. `txtCostCenter_1` is a free-text `ModernTextInput` (no dropdown/Items list at all — the earlier hard-coded 6-warehouse-name combo was replaced), pre-filled from the selected Project but user-editable.

**`RequestFormScreen` has a required `ddProject_1` picker** (`Project_List`, sourced directly — this app must have `Project_List` added as a data source in Studio, same site but a separate Power Apps canvas app from `project-list`). Every new procurement request must be tied to a project — submit is blocked with "Please select the Project this request is for" until `ddProject_1.Selected` is non-blank, checked *before* the rest of the required-field validation.

Once a project is selected, it auto-fills `Department` (`ddDepartment_1`, a `ComboBox` via `DefaultSelectedItems`), `Currency` (`txtCurrency_1`, a free-text `ModernTextInput` via `Default: =ddProject_1.Selected.Currency`), and `Cost Center` (`txtCostCenter_1`, same pattern via `Default: =ddProject_1.Selected.CostCenter`). All 3 fields are then **read-only** — `DisplayMode: =If(IsBlank(ddProject_1.Selected), DisplayMode.Disabled, DisplayMode.View)` — disabled until a project is chosen, then view-only (auto-filled value shown, not user-editable) once it is. The user cannot override them manually; changing `ddProject_1` recomputes and re-displays all 3. `Cost Center`→`InvoiceRegion` still goes through the country lookup `Switch` (unchanged, now reading `txtCostCenter_1.Text` instead of a combo's `.Selected.Value`) on top of this.

`txtCurrency_1` sits in the same row/group as `txtEstimatedCost_1` (`colEstimatedCostRow`, mirroring `project-list/CreateProjectScreen.pa.yaml`'s `colBudgetAmountRow` pattern: the amount field takes `FillPortions: =1`, the currency field is a small fixed-width sibling) — replaces the old separate `colCurrency` column and the `ddCurrency_1` ComboBox.

| Column | Type | Notes |
|---|---|---|
| `Tiêu đề` (Title) | Text | Auto-built: `<employee> - <Category> - <dd/mm/yyyy>` |
| `Status` ⚠ | Choice | **Drives the workflow** — value list below |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `ProjectID` ⚠ | Text | **New, not yet created on SharePoint** — the related `project-list` project's business key (`Project_List.ProjectID`, e.g. `PROJ-AU-QA-2026-017`), set from the required `ddProject_1` picker on `RequestFormScreen` (every request must be tied to a project — see picker note below). Also the join column the future `Project_SyncActualCost` flow was already waiting on (see `project-list/CLAUDE.md` §Next) |
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
| `Currency` ⚠ | Text | Free-typed via `txtCurrency_1` (pre-filled from the selected Project's `Currency`, e.g. `AUD`/`MYR`/`SGD`/`VND`) — **not** a Choice column, same reasoning as `CostCenter` below (just passed through, no cross-app Choice value set to sync) |
| `BudgetReference` | Text | |
| `RequiredDeliveryDate` ⚠ | Date | |
| `DeliveryLocation` ⚠ | Choice | Independent from `CostCenter` below — delivery destination for goods, not the warehouse/office used for invoice filing |
| `CostCenter` ⚠ | Text | **New, not yet created on SharePoint** — plain Text (not Choice), filled directly from `project-list`'s `Project_List.CostCenter` (also Text) without keeping two Choice value sets in sync. Free-text field on `RequestFormScreen` (`txtCostCenter_1`, a `ModernTextInput` — no dropdown/Items list), pre-filled from the selected Project but user-editable. `InvoiceRegion` (below) is auto-derived from it via a `Switch` lookup table, not separately picked |
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
| `GRAssignedToID` | Lookup→Employee List | delegate for Step 3 Goods Receipt; blank = Requester performs it |
| `SFU1AssignedToID` | Lookup→Employee List | delegate for Step 4 Supplier Follow-up (Requester); blank = Requester performs it |
| `GoodsReceiptBy` | Text | |
| `GoodsReceiptDate` | DateTime | |
| `GoodsReceiptStatus` | Choice | |
| `GoodsAcceptanceDecision` | Choice | `Accepted`, `Rejected`, `Requires Supplier Follow-up` |
| `GoodsReceiptRemarks` | Text (multiline) | |
| `GoodsReceiptAt` | DateTime | |
| `FollowUpReceiptBy` | Text | |
| `FollowUpReceiptDate` | DateTime | |
| `FollowUpReceiptStatus` | Choice | same choices as `GoodsReceiptStatus` |
| `FollowUpAcceptanceDecision` | Choice | `Accepted`, `Accepted with Adjustment` |
| `CreditNote` | Text | required when follow-up decision = adjustment |
| `Fulfillment` | Choice | `Fulfilled`, `Fulfilled with Adjustment` |
| `FollowUpRemarks` | Text (multiline) | |
| `FollowUpReceiptAt` | DateTime | |
| `SupplierFollowUpNotes` | Text (multiline) | |
| `FollowUpCompletedAt` | DateTime | |
| `InvoiceRegion` ⚠ | Choice | Country code (`AU`/`MY`/`SG`) used to file the invoice into the correct storage folder. **No longer directly user-selected** — auto-derived on submit from `CostCenter` (above) via a `Switch` lookup table in `RequestFormScreen.pa.yaml` (warehouse/office → country) |
| `Attachments` | Attachments | invoice/supporting files; written via `Form1`+`SubmitForm` (Patch can't write attachments) |

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
