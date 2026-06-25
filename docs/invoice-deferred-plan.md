# Invoice Deferred Flow — Implementation Plan

## Background

The current procurement flow is strictly sequential:
**Pending Procurement → Pending Accounting → Goods Receipt → Completed**

This breaks down in practice because supplier invoices often arrive late — sometimes after
goods have already been received. Accounting should only be involved when a final official
invoice exists. This plan introduces a deferred-invoice path and an intermediary path where
the Requester works directly with the Supplier.

---

## New Status

| Status | Meaning |
|---|---|
| `Pending Invoice` | Invoice was deferred at Procurement stage. Waiting for Procurement (or Requester in Via Requester path) to provide the invoice before Accounting can review. |

---

## Three Procurement Paths

### Path A — Invoice available immediately (existing behavior, unchanged)
```
Pending Procurement → [submit invoice] → Pending Accounting → Goods Receipt → Completed
```

### Path B — Procurement buys directly, invoice arrives late
```
Pending Procurement → [Defer Invoice] → Goods Receipt & Acceptance
  → [GR done] → Pending Invoice
  → InvoiceSubmissionScreen (Procurement, AI parse) → Pending Accounting → Completed
```

### Path C — Procurement is intermediary, Requester works directly with Supplier
```
Pending Procurement → [Upload ủy nhiệm chi + Submit] → email to Requester
  → Goods Receipt & Acceptance (Requester attaches supplier invoice here)
  → [GR done] → Pending Invoice
  → InvoiceSubmissionScreen (Procurement views Requester's invoice, AI parse) → Pending Accounting → Completed
```

If GR outcome is "Requires Supplier Follow-up", the invoice attachment field also appears
at SupplierFollowUpScreen (if not already submitted at GR stage).

---

## Routing Logic After GR / SFU

| Screen | Outcome | InvoiceMode | Next Status |
|---|---|---|---|
| GoodsReceiptScreen | Accepted | `Direct` | Completed |
| GoodsReceiptScreen | Accepted | `Deferred` or `ViaRequester` | **Pending Invoice** |
| GoodsReceiptScreen | Requires Follow-up | `Deferred` or `ViaRequester` | Pending Supplier Follow-up |
| SupplierFollowUpScreen | Accepted | `Direct` | Completed |
| SupplierFollowUpScreen | Accepted | `Deferred` or `ViaRequester` | **Pending Invoice** |

---

## Task Breakdown

### Task 1 — SharePoint Schema
**File to reference:** `docs/sharepoint-schema.md`

Add 3 new columns to `Procurement_Requests`:

| Column | Type | Values / Notes |
|---|---|---|
| `InvoiceMode` | Choice | `Direct` / `Deferred` / `ViaRequester` — set at Procurement stage |
| `PaymentAuthorizationURL` | Text (single line) | URL to uploaded ủy nhiệm chi file (Path C only) |
| `RequesterInvoiceURL` | Text (single line) | URL to invoice attached by Requester at GR or SFU stage (Path C only) |

---

### Task 2 — ProcurementExecutionScreen
**File:** `Src/ProcurementExecutionScreen.pa.yaml`

Replace the current single submit path with a 3-way choice:

**UI changes:**
- Add a radio/segmented control or 3 buttons:
  - `Submit Invoice Now` → existing AI parse + submit flow (Path A)
  - `Defer Invoice` → confirm button, no invoice needed (Path B)
  - `Send Payment Authorization` → file upload field for ủy nhiệm chi (Path C)

**Logic — Path B (Defer):**
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "Deferred",
    Status: "Goods Receipt & Acceptance"
})
```
Write `Procurement_ExecutionLog` row (StepNumber 1) without invoice data.

**Logic — Path C (Via Requester):**
- Show file upload control → on upload success, store URL in `PaymentAuthorizationURL`
- On submit:
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "ViaRequester",
    PaymentAuthorizationURL: <uploaded file URL>,
    Status: "Goods Receipt & Acceptance"
})
```
- Trigger Power Automate flow to email Requester with ủy nhiệm chi link.

---

### Task 3 — GoodsReceiptScreen
**File:** `Src/GoodsReceiptScreen.pa.yaml`

**UI changes:**
- Add invoice attachment section, visible only when:
  ```
  gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL)
  ```
- Section contains: file upload control + label explaining Requester must attach supplier invoice.

**Logic — on submit GR:**
- If invoice attachment is present, patch `RequesterInvoiceURL` first.
- Determine next status:
  ```
  If(
      gSelectedRequest.InvoiceMode = "Direct",
      /* existing routing: Completed / Rejected / Pending Supplier Follow-up */,
      /* Deferred or ViaRequester: */
      If(
          grOutcome = "Requires Follow-up",
          "Pending Supplier Follow-up",
          "Pending Invoice"   // replaces Completed when invoice is still owed
      )
  )
  ```
- If `RequesterInvoiceURL` was just set: trigger notification flow to Procurement.

---

### Task 4 — SupplierFollowUpScreen
**File:** `Src/SupplierFollowUpScreen.pa.yaml`

Same pattern as Task 3, applied to Step 1 (Requester detail step):

**UI changes:**
- Add invoice attachment section with same visibility condition:
  ```
  gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL)
  ```

**Logic — on submit SFU Step 1:**
- If attachment present, patch `RequesterInvoiceURL`.
- Trigger notification flow to Procurement if invoice was just submitted.

**Logic — on submit SFU Step 2 (Procurement finalizes):**
- Accepted + `InvoiceMode ≠ "Direct"` → Status = `"Pending Invoice"` (not Completed).
- Accepted + `InvoiceMode = "Direct"` → Status = `"Completed"` (unchanged).

---

### Task 5 — InvoiceSubmissionScreen (new screen)
**File:** `Src/InvoiceSubmissionScreen.pa.yaml` (new, based on ProcurementExecutionScreen)

**Purpose:** Procurement processes the deferred invoice after GR/SFU is done.

**Triggered from:** HomeScreen when request status = `Pending Invoice` and user role = Procurement.

**UI:**
- If `InvoiceMode = "ViaRequester"` and `RequesterInvoiceURL` is set:
  show a read-only link/preview so Procurement can view Requester's attached invoice.
- Same AI parse flow as ProcurementExecutionScreen (Parse_Invoice + Submit_Invoice flows).
- No defer option — invoice must be submitted here.

**Logic — on submit:**
```
Patch(Procurement_Requests, gSelectedRequest, {
    Status: "Pending Accounting",
    // invoice fields same as ProcurementExecutionScreen
})
```
Write `Procurement_ExecutionLog` row (use a new StepNumber, e.g. StepNumber 6, or reuse 1
with a flag — decide at implementation time).

---

### Task 6 — HomeScreen
**File:** `Src/HomeScreen.pa.yaml`

**Changes to Procurement filter:**
- Add `"Pending Invoice"` to the list of statuses Procurement sees in their gallery Items filter.

**Changes to Requester filter:**
- Requester currently sees their own requests. For requests where
  `InvoiceMode = "ViaRequester"` and status = `"Pending Invoice"`, Requester should still
  be able to see the request (read-only, to track progress). Verify this is covered by
  the existing `RequesterEmail = gCurrentEmployee.Email` filter — it should be, no
  extra change needed unless the filter explicitly excludes certain statuses.

**Filter button:**
- Add `"Pending Invoice"` as a filter button in the status chip row (visible to Procurement role).

---

### Task 7 — Power Automate Flows

**Flow 7a — Notify Requester: Payment Authorization Ready**
- Trigger: called from ProcurementExecutionScreen (Path C submit)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `paymentAuthUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with ủy nhiệm chi download link and instruction to collect invoice from supplier.

**Flow 7b — Notify Procurement: Invoice Provided by Requester**
- Trigger: called from GoodsReceiptScreen or SupplierFollowUpScreen when `RequesterInvoiceURL` is set on submit
- Parameters: `procurementEmail`, `procurementName`, `requestTitle`, `requestId`, `requesterInvoiceUrl`
- Action: send Outlook email + Teams Adaptive Card to Procurement notifying that Requester has attached the invoice and it is ready to process.

> Both flows can be created as new standalone flows or added as new `notificationType`
> values to the existing `Procurement_Notify_Receipt_Assignee` flow if the parameter
> sets are compatible.

---

## Implementation Order

```
Task 1 (SharePoint) → Task 2 (ProcurementExecutionScreen)
                    → Task 6 (HomeScreen)
                    → Task 3 (GoodsReceiptScreen)
                    → Task 4 (SupplierFollowUpScreen)
                    → Task 5 (InvoiceSubmissionScreen)
                    → Task 7 (Power Automate flows)
```

Tasks 2–6 can proceed after Task 1. Tasks 3 and 4 are independent of each other.
Task 5 and Task 7 can be done in parallel with Tasks 3–4.

---

## Decisions

**InvoiceSubmissionScreen StepNumber in `Procurement_ExecutionLog`:** **StepNumber 6**
Distinguishes the deferred invoice submission from the initial procurement action (StepNumber 1).
Look up the deferred invoice log row with:
```
LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 6)
```
