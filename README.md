# Max Biocare Procurement Procedure

Power Apps Canvas App (Power Fx) managing the end-to-end procurement request workflow for Max Biocare.

This repository tracks the **screen logic and UI only** (`Src/*.pa.yaml`) for change history and review. It is **not** a full `.msapp` package — the generated metadata (`Controls/`, `References/`, `Properties.json`, …) is intentionally excluded (see [`.gitignore`](.gitignore)). The source of truth for editing remains **Power Apps Studio**; this repo is for tracing changes over time.

## Repository layout

```
Src/
├── _EditorState.pa.yaml          # Screen order
├── App.pa.yaml                   # App.OnStart — user resolution + global state
├── HomeScreen.pa.yaml            # Role-based request list + status filters
├── RequestFormScreen.pa.yaml     # Requester submits a new request
├── RequestDetailScreen.pa.yaml   # Read-only detail + history of a request
├── ManagerReviewScreen.pa.yaml   # Step 2 — department manager decision
├── ExecutiveApprovalScreen.pa.yaml   # Step 3 — executive decision
├── ProcurementExecutionScreen.pa.yaml # Procurement + invoice parsing
├── AccountingScreen.pa.yaml      # Accounting handling
├── GoodsReceiptScreen.pa.yaml    # Goods receipt & acceptance
└── SupplierFollowUpScreen.pa.yaml    # Supplier follow-up (2 steps)
```

See [`CLAUDE.md`](CLAUDE.md) for the detailed architecture and conventions.

## Backend

- **SharePoint Online** (`maxbiocare.sharepoint.com/sites/Powerapps`) — lists: `Procurement_Requests` (central record), `Procurement_User` (roles), `Procurement_ApprovalLog`, `Procurement_ExecutionLog`, `Procurement_InvoiceData`, `Suppliers`, `Employee List`. Full column reference: [`docs/sharepoint-schema.md`](docs/sharepoint-schema.md).
- **Power Automate** flows called from Power Fx: `Parse_Invoice` (AI invoice extraction) and `Submit_Invoice`.

## Workflow

Requests move through the `Status` choice field on `Procurement_Requests`:

```
New request (RequestFormScreen)
  │  Auto-skip to Executive if PurchaseAccordance = "Urgent"/"Unplanned"
  │  OR (EstimatedCost > 5000 AND Currency = "AUD")  → SkippedManagerReview
  ▼
Pending Manager   → "Approved (within budget)" → Pending Procurement
                  → otherwise                   → Pending Executive
Pending Executive → Reject → Rejected
                  → Approve / Approve with conditions → Pending Procurement
Pending Procurement → Pending Accounting   (Parse_Invoice / Submit_Invoice; or Rejected)
Pending Accounting  → Goods Receipt & Acceptance
Goods Receipt & Acceptance → Accepted                    → Completed
                           → Rejected                    → Rejected
                           → Requires Supplier Follow-up → Pending Supplier Follow-up
Pending Supplier Follow-up → (Step 1: Requester, Step 2: Procurement) → Completed
```

## Roles

`Requester` · `Manager` · `Executive` · `Procurement` · `Accounting` · `Admin` — resolved from `Procurement_User`; `HomeScreen` filters the request list per role. Users absent from `Employee List` are denied access.

## Editing / sync workflow

1. Edit the app in **Power Apps Studio**.
2. Export and unpack with the Power Platform CLI: `pac canvas unpack --msapp <file>.msapp --sources <this-repo>`.
3. The unpack regenerates `Controls/`, `References/`, etc. — these are git-ignored; commit only the `Src/` changes.

## Conventions

- All UI text, field names, comments, and code are **English**. Vietnamese appears only as SharePoint system-column internal names (`'Tiêu đề'` = Title, `'Tệp đính kèm'` = Attachments — quote them in Power Fx) and the `vi-VN` locale.
- Status strings are repeated as literals across screens (no shared constant) — when changing the flow, update every screen that references the string.
