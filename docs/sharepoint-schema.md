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

Required ⚠: `Status`, `RequesterEmail`, `ProcurementType`, `ProcurementDescription`, `Category`, `PurchaseAccordance`, `EstimatedCost`, `Currency`, `RequiredDeliveryDate`, `DeliveryLocation`, `RequesterID`, `InvoiceRegion`, `CostCenter`.

**Not yet created on SharePoint**: `RequirementFiles` (**Multiple lines of text**, plain) — holds the `|`-delimited file names uploaded through `RequestFormScreen`'s "Requirement Files" box. Until it exists the create `Patch` fails and `RequestDetailScreen`'s "Requirement Files (Requester)" list stays empty. Plain multi-line rather than single-line because ten file names easily exceed 255 characters. See `colRequirementFiles` / `gPendingRequestFiles` in `CLAUDE.md`. Also `CostCenter` (**Choice**, same 6 values as `project-list`'s `Project_List.CostCenter` — see screenshot-derived list in the row below). `ProjectID` (Text) **does now exist on the live list** (confirmed 2026-07-27). **`Department` and `Currency` both stay their existing live types, no SharePoint change needed** — `Department` is (and always was) plain Text, picker options sourced from `Choices('Employee List'.Department)`, same convention as `project-list`'s own `Project_List.Department`. `Currency` is confirmed **Text** on the live list (Studio's own type-check settled this: writing `{Value: ...}` to it threw `expected Text, found Record` — so it's Text, not Choice as an earlier pass here assumed).

**Also not yet created**: `ManagerApproverNum`, `GRAssignedToNum`, `SFU1AssignedToNum` — all plain **Number** columns, one per existing Lookup column (`ManagerApproverID`, `GRAssignedToID`, `SFU1AssignedToID` respectively). Each holds the same `.Id` value as its Lookup counterpart and is written at the exact same time (`RequestFormScreen`'s create `Patch` for `ManagerApproverNum`; `GoodsReceiptScreen`'s assign/unassign/round-clear `Patch`es for `GRAssignedToNum`; `SupplierFollowUpScreen`'s equivalents for `SFU1AssignedToNum` — blank whenever the Lookup one is blank). They exist purely so `HomeScreen` can filter `Procurement_Requests` by "is this employee the approver/assignee" with a **delegable** Number comparison — comparing a Lookup/Person column's `.Id` directly in a `Filter` against a remote SharePoint table is not delegable and triggers Studio's "might not work correctly on large data sets" warning, which is what these mirror columns exist to avoid. Until they exist on the live list, `HomeScreen`'s `colHomeRoleFiltered` (`OnVisible`) and `galRequests.Items` — the only two places that filter on these fields against the live table — will fail to find any matching rows for the Manager/GR-assignee/SFU-assignee branches (the write-side `Patch`es will also fail, since a `Patch` targeting a non-existent column is a hard error, not a silent no-op). Everywhere else in the app that reads `ManagerApproverID.Id`/`GRAssignedToID.Id`/`SFU1AssignedToID.Id` off an already-fetched single record (`ThisItem`, `gSelectedRequest`, `gQuickViewRequest`) is unaffected and keeps using the Lookup column directly — delegation only applies to formulas that query the remote table.

**Linking a request to a Project is optional.** `RequestFormScreen` has a `rdoHasProject` Radio ("Related to a Project?", `Items: =["No", "Yes"]` — defaults to "No", first item) gating the `ddProject_1` picker (`Project_List`, sourced directly — this app must have `Project_List` added as a data source in Studio, a separate Power Apps canvas app from `project-list`, same site). `ddProject_1` and its label/lookup-link are only `Visible` when `rdoHasProject.Selected.Value = "Yes"`, and submit only requires it in that case: `If(rdoHasProject.Selected.Value = "Yes" && IsBlank(ddProject_1.Selected), <error>, ...)`.

When `rdoHasProject = "Yes"` and a project is selected, `Department` (`ddDepartment_1`) and `Cost Center` (`ddCostCenter_1`) — both ordinary `ComboBox` controls — auto-fill via `DefaultSelectedItems: =Filter(Choices(<source>), rdoHasProject.Selected.Value = "Yes" && Value = ddProject_1.Selected.<ProjectField>)` and go **read-only**: `DisplayMode: =If(Not(rdoHasProject.Selected.Value = "Yes"), DisplayMode.Edit, If(IsBlank(ddProject_1.Selected), DisplayMode.Disabled, DisplayMode.View))`. When `rdoHasProject = "No"` (or "Yes" but no project chosen yet), both stay normal editable dropdowns — the user picks manually, matching Project's own value sets so the two apps stay conceptually aligned even when not linked. `<source>` differs per field: `ddDepartment_1` sources `Choices('Employee List'.Department)` (Department itself is plain Text on `Procurement_Requests`, not a Choice column — options only, same as `project-list`); `ddCostCenter_1` sources `Choices(Procurement_Requests.CostCenter)` (a real Choice column here).

`Currency` is an editable `ComboBox` (`ddCurrency_1`, `Items: =["MYR", "SGD", "AUD", "VND", "CNY", "USD"]`, sits beside `txtEstimatedCost_1` in `colEstimatedCostRow`, mirroring `project-list/CreateProjectScreen.pa.yaml`'s `colBudgetAmountRow` layout), not a plain label anymore. It auto-jumps whenever `Cost Center` changes — via `DefaultSelectedItems: =If(IsBlank(ddCostCenter_1.Selected), Blank(), Table({Value: Switch(ddCostCenter_1.Selected.Value, "Office Melbourne (Head Quarter)", "AUD", "Port Melbourne Warehouse", "AUD", "Max Biocare Research Park - Natural Inspirations@Yinnar", "AUD", "Max Biocare Research Park - Mar-Nuka Bay", "AUD", "Malay Warehouse", "MYR", "Singapore Warehouse", "SGD")}))` plus `ddCostCenter_1.OnChange: =Reset(ddCurrency_1)` to force the jump even after a manual override — but the user can still pick a different currency manually afterward if needed. Same warehouse-to-region mapping as `InvoiceRegion` below, just a currency code instead of a country code — kept as a separate duplicated `Switch` (this app's convention: no shared constants) since Patch also needs it standalone. Patch writes `Currency: ddCurrency_1.Selected.Value` directly (no re-computing the `Switch` at submit time), and submit validation now requires `IsBlank(ddCurrency_1.Selected)` alongside the other required fields.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | Auto-built: `<employee> - <Category> - <dd/mm/yyyy>` |
| `Status` ⚠ | Choice | **Drives the workflow** — value list below |
| `RequesterEmail` ⚠ | Text | `User().Email`; "my requests" filter key |
| `ProjectID` | Text | Exists on the live list (confirmed 2026-07-27) — the related `project-list` project's business key (`Project_List.ProjectID`, e.g. `PROJ-AU-QA-2026-017`), set from `ddProject_1` when `rdoHasProject = "Yes"`; blank when the request isn't tied to a project. Also the join column the future `Project_SyncActualCost` flow was already waiting on (see `project-list/CLAUDE.md` §Next) |
| `Department` ⚠* | Text | Plain text (\*not required in schema, but set on submit) — picker options from `Choices('Employee List'.Department)`, same convention as `project-list`. `ddDepartment_1` auto-fills/read-only from the selected Project's `Department` when linked, otherwise a normal editable dropdown |
| `ProcurementType` ⚠ | Choice | incl. `Invoice Supplied` |
| `InvoiceType` | Choice | incl. `Official Invoice`; blank unless ProcurementType = Invoice Supplied |
| `InvoiceLink` | Text (URL) | Original/uploaded invoice link |
| `OfficialInvoiceLink` | Text (URL) | Final official invoice link set by Procurement |
| `ProcurementDescription` ⚠ | Text (multiline) | |
| `Category` ⚠ | Choice | |
| `PreferredSupplier` | Text | Supplier title |
| `PurchaseAccordance` ⚠ | Choice | incl. `Urgent`, `Unplanned` (auto-escalate to Executive) |
| `EstimatedCost` ⚠ | Number | |
| `Currency` ⚠ | Text | Confirmed Text (not Choice) via Studio's type-check. Editable `ddCurrency_1` ComboBox offering `MYR`/`SGD`/`AUD`/`VND`/`CNY`/`USD` — auto-jumps to `AUD`/`MYR`/`SGD` from `Cost Center` on change (see note above; the `Switch` never yields `VND`/`CNY`/`USD`, those are manual-only picks) but can be manually overridden to any of the six; written as a plain string on submit, no `{Value:}` wrapper |
| `BudgetReference` | Number | Optional. `txtBudgetRef_1` is still a plain `TextInput`, so `btnSubmit_1` rejects a non-numeric entry and the create `Patch` writes `If(IsBlank(…), Blank(), Value(…))` — a bare `Value("")` on the empty (allowed) case would error against a Number column. `RequestDetailScreen` renders it like `EstimatedCost`: `"#,###.00"` + `Currency`, falling back to `"—"` when blank |
| `RequiredDeliveryDate` ⚠ | Date | |
| `DeliveryLocation` ⚠ | Choice | Independent from `CostCenter` below — delivery destination for goods, not the warehouse/office used for invoice filing |
| `CostCenter` ⚠ | Choice | **New, not yet created on SharePoint** — same 6 values as `project-list`'s `Project_List.CostCenter`: `Office Melbourne (Head Quarter)`, `Port Melbourne Warehouse`, `Max Biocare Research Park - Natural Inspirations@Yinnar`, `Max Biocare Research Park - Mar-Nuka Bay`, `Malay Warehouse`, `Singapore Warehouse`. `ddCostCenter_1` auto-fills/read-only from the selected Project's `CostCenter` when linked, otherwise a normal editable dropdown sourced from `Choices(Procurement_Requests.CostCenter)`. `InvoiceRegion` is auto-derived from it via a `Switch` lookup, never separately picked; `Currency` (`ddCurrency_1`) auto-jumps the same way but is a separate editable ComboBox the user can override |
| `RequesterID` ⚠ | Lookup→Employee List | `{Id,Value}` (→Title) |
| `ManagerApproverID` | Lookup→Employee List | blank when manager review skipped |
| `ManagerApproverNum` | Number | **New, not yet created on SharePoint** — plain numeric mirror of `ManagerApproverID.Id`, written alongside it, purely so `HomeScreen` can filter by it delegably (see note above) |
| `SkippedManagerReview` | Yes/No | true when Urgent/Unplanned **or** (EstimatedCost > 5000 AND Currency = AUD) |
| `isExecutivePayment` | Yes/No | true when Executive approves an over-threshold request (`Currency <> "AUD" \|\| EstimatedCost > 10000`, a *different* threshold from `SkippedManagerReview`'s 5000). Set by `ExecutiveApprovalScreen` on Approve/Approve-with-conditions; `Status` stays `"Pending Executive"` while this is true (UI shows "Pending Payment From Executive" as a computed label — see `CLAUDE.md`). Cleared back to false only implicitly never — stays true for history once the request has moved on |
| `RemittanceURL` | Text (URL) | Proof-of-payment attachment link. Two independent producers: `ExecutivePaymentScreen` (when `isExecutivePayment = true`) and `ProcurementExecutionScreen`'s own Path C "Remittance Advice Document" upload (`locIsViaRequester`). `ProcurementExecutionScreen` skips its own upload requirement and reuses this field's existing value when `isExecutivePayment = true` — see `CLAUDE.md` |
| `ProcurementExecutedBy` | Lookup→Employee List | |
| `ProcurementExecutedAt` | DateTime | |
| `AccountingHandlerID` | Lookup→Employee List | **who completed the accounting step**, not who it was assigned to — written only by `AccountingScreen`'s submit as `gCurrentEmployee`, so it stays blank until the request is `Completed` |
| `AccountingCompletedAt` | DateTime | |
| `ConditionsText` | Text (multiline) | set on "Approve with conditions" |
| `RequestID` | Lookup (→ID) | present in schema; not used by the app flow |
| `PurchaseRequestLink` | Text (URL) | |
| `UsageExpiryDate` | Date | **Limited-use goods flag** — the last date the goods are usable (e.g. event materials). Set by Procurement on `ProcurementExecutionScreen` (`rdoLimitedUse_PE` = Yes → `dpLimitedUseDate_PE`); blank = normal goods, no restriction. There is deliberately **no separate Yes/No column**: non-blank *is* the flag. Read-only afterwards — shown as a "Do Not Store" badge on `HomeScreen` and a red panel on `RequestDetailScreen` so warehouse staff know the goods must not be put into storage past this date |
| `GRAssignedToID` | Lookup→Employee List | delegate for Step 3 Goods Receipt; blank = Requester performs it |
| `GRAssignedToNum` | Number | **New, not yet created on SharePoint** — plain numeric mirror of `GRAssignedToID.Id`, written alongside it, purely so `HomeScreen` can filter by it delegably (see note above) |
| `SFU1AssignedToID` | Lookup→Employee List | delegate for Step 4 Supplier Follow-up (Requester); blank = Requester performs it |
| `SFU1AssignedToNum` | Number | **New, not yet created on SharePoint** — plain numeric mirror of `SFU1AssignedToID.Id`, written alongside it, purely so `HomeScreen` can filter by it delegably (see note above) |
| `ReceiptRoundCount` ✳ | Number | **new** — receipt rounds submitted so far; the round being entered next is `+1`. Read as `Coalesce(…, 0)` everywhere so a request that hasn't reached Goods Receipt reads `0` rather than blank. In practice only ever `0`, `1` or `2` — deliveries are capped at two rounds by the round-2 dropdown, not by this column |
| `LatestReceiptDecision` ✳ | Text | **new** — the most recent round's acceptance decision. **Plain Text, not Choice**, so both the round-1 and round-N option sets fit in one column |
| `CreditNote` | Text | required when follow-up decision = adjustment |
| `Fulfillment` | Choice | `Fulfilled`, `Fulfilled with Adjustment` |

**Deleted with the unlimited-rounds change** (no legacy read path is left in the app — it will not compile if any of these is still referenced, and will not run correctly until the two columns above exist): `GoodsReceiptBy`, `GoodsReceiptDate`, `GoodsReceiptStatus`, `GoodsAcceptanceDecision`, `GoodsReceiptRemarks`, `GoodsReceiptAt`, `FollowUpReceiptBy`, `FollowUpReceiptDate`, `FollowUpReceiptStatus`, `FollowUpAcceptanceDecision`, `FollowUpRemarks`, `FollowUpReceiptAt`, `SupplierFollowUpNotes`, `FollowUpCompletedAt`. Two fixed column sets could only ever hold two rounds; per-round data now lives in `'Procurement Receipt Rounds'`. The four receipt Choice columns are gone too — their dropdowns hold the options as **inline literal tables** in Power Fx now (`["Fully Received", …]`), so changing an option is a code edit, and renaming one also means updating every `= "..."` comparison in the receiving loop.
| `InvoiceRegion` ⚠ | Choice | Country code (`AU`/`MY`/`SG`) used to file the invoice into the correct storage folder. **Not directly user-selected** — auto-derived on submit from `ddCostCenter_1.Selected.Value` via a `Switch` lookup table in `RequestFormScreen.pa.yaml` (warehouse/office → country) |
| `Attachments` | Attachments | invoice/supporting files; written via `Form1`+`SubmitForm` (Patch can't write attachments) |

### `Status` choice values (exact strings — used as literals across all screens)

`Pending Manager` → `Pending Executive` → `Pending Procurement` → `Pending Accounting` → `Goods Receipt & Acceptance` → `Pending Supplier Follow-up` → `Completed`, plus terminal `Rejected`.

> Changing any string means updating every screen's filters, color maps, and `Patch`/`If`/`Switch` logic — there is no shared constant.

---

## `Procurement_User` — role assignment

Required ⚠: `Role`, `IsActive`, `EmployeeID`.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | |
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
| `Title` (Title) | Text | `Step <n> - <decision> - <request title>` |
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
| `Title` (Title) | Text | `Step <n> - <name> - <request title>` |
| `RequestID` | Lookup (→ID) | `{Id,Value}` |
| `RequestIDText` | Text | join key |
| `StepNumber` | Number | `1` Procurement Execution · `2` Accounting Handover · `3` Goods Receipt **round 1** · `4` Goods Receipt **round N ≥ 2** · `5` Supplier Follow-up (Procurement close-out) · `6` Invoice Submission |
| `StepName` | Choice | matches the step |
| `ExecutedBy` | Lookup→Employee List | |
| `ExecutedAt` | DateTime | |
| `HandoverToID` | Lookup→Employee List | **orphaned** — held the Accounting staffer picked ahead of time on `ProcurementExecutionScreen` / `InvoiceSubmissionScreen`; both pickers were removed (see "Accounting is a shared queue" in `CLAUDE.md`), so nothing writes or reads this. Safe to delete |
| `HandoverToIDText` | Text | **orphaned**, same reason |
| `SupplierSummary` | Text (multiline) | step 1 |
| `PurchaseOrderLink` | Text (URL) | step 1 |
| `Notes` | Text (multiline) | |

Steps 1/2/6 are unique per request and can still be `LookUp`ed. **Steps 3/4/5 are not** — there is one row per receipt round, so filter and sort them instead. There can be many step-4 rows; a step-5 row is written at most once, on the `Accepted with Adjustment` branch.

Receipt data (who received, when, status, decision, remarks, photos) is **not** on this list — it lives on `'Procurement Receipt Rounds'` below. The log keeps only the generic step trail so the same header isn't stored twice.

---

## `'Procurement Receipt Rounds'` — one row per receipt round ✳ **new list**

Required ⚠: `RequestIDText`, `RoundNumber`.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | `R<n> - <request title>` |
| `RequestID` | Lookup (→ID) | `{Id,Value}` |
| `RequestIDText` | Text | join key — every read filters on this |
| `RoundNumber` | Number | 1 for the Goods Receipt round, 2+ for each follow-up round |
| `ReceivedBy` | Text | |
| `ReceiptDate` | Date | |
| `ReceiptStatus` | Text | `Fully Received` · `Partially Received` · `Incorrect Items` · `Damaged Items` |
| `AcceptanceDecision` | Text | round 1: `Accepted` · `Rejected` · `Requires Supplier Follow-up`; round 2: `Accepted` · `Accepted with Adjustment` only — round 2 has no option that opens a round 3, which is what caps deliveries at two |
| `Remarks` | Text (multiline) | |
| `Attachments` (system) | Attachments | **the round's receipt photos live here.** `frmGRLog_GR` / `frmSFU1Log_SFU` bind to *this* list, which is why each submit `Patch`es the round row **first** and only then submits the form against it |

**One row per round, not per material** — this app has no line-item list, so a round is a single record. That is also why the round header lives here and **not** on `Procurement_ExecutionLog`: in the sibling `procurement-raw-materials` app the two lists carry different grains (log = one row per round header, rounds list = one row per *line item × round*), so nothing is duplicated. Without line items that split has no second grain to justify it, so this list owns everything about receiving — header, remarks and photos — and the execution log keeps only its generic step trail (`ExecutedBy`, `ExecutedAt`, `Notes`). **Don't add `RoundNumber`/`ReceivedBy`/`ReceiptDate`/`ReceiptStatus`/`AcceptanceDecision` to the log** — that was an earlier draft of this change and it made every submit write the same header twice.

The Procurement close-out (Credit Note) is **not** a receipt round and writes no row here — it is log step 5 only, and shows up in `RequestDetailScreen`'s Execution History rather than its Goods Receipt table.

---

## `Procurement_InvoiceData` — extracted invoice fields

Required ⚠: none. **Not written by app `Patch`** — populated by the `Submit_Invoice` flow (see below). Column names differ from the app's variable names — note `VendorName` and `GSTAmount`.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | |
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
| `Title` (Title) | Text | employee display name |
| `email` | Text | matched against `User().Email` in `App.OnStart` |
| `department` | Choice | |
| `JobTitle` | Text | |
| `City` | Text | |
| `Country` | Text | |
| `EmployeeCode` | Text | |

---

## `Suppliers`

Required ⚠: none. Bound to the "Preferred Supplier" picker via `Sort(Suppliers, 'Title')`.

| Column | Type | Notes |
|---|---|---|
| `Title` (Title) | Text | supplier name |
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

Both are called from `ProcurementExecutionScreen` (and `Submit_Invoice` also from `InvoiceSubmissionScreen`) via `<Flow>.Run(...)`.

### `Parse_Invoice.Run(invoiceLink, requestId)` — AI invoice extraction

| Input | Type | |
|---|---|---|
| `text` | string | invoice link |
| `number` | number | request ID |

**Returns** (`gInvoiceResult`): `invoiceNumber`, `invoiceDate`, `totalAmount` (n), `taxAmount` (n), `supplierName`, `supplierABN`, `billedTo`, `currency`, `confidenceScore` (n), `jobId`, `attention`, `suggestedFilename`.

### `Submit_Invoice.Run(...)` — writes the official invoice + `Procurement_InvoiceData`

18 positional args (the trigger names them `text`, `number`, `text_1`…); the app passes them in this order at all three call sites (`ProcurementExecutionScreen` Path A / Path C-Direct, `InvoiceSubmissionScreen`):

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
| 14 | `text_9` | invoice **Description** field value |
| 15 | `text_10` | invoice region |
| 16 | `text_11` | ABN |
| 17 | `text_12` | source app name (`"Procurement App"`) |
| 18 | `text_13` | `gSelectedRequest.ProjectID` |

**Returns**: `newinvoicelink` (string).
