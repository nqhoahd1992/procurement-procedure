# Invoice Deferred Flow — Implementation Plan

## Background

The current procurement flow is strictly sequential:
**Pending Procurement → Pending Accounting → Goods Receipt → Completed**

This breaks down in practice because supplier invoices often arrive late — sometimes after
goods have already been received. Accounting should only be involved when a final official
invoice exists. This plan introduces a deferred-invoice path and an intermediary path where
the Requester works directly with the Supplier.

**Design principle:** `Pending Accounting` is always the final step before `Completed`, for
every path. GR and Supplier Follow-up always happen before Accounting.

---

## New Status

| Status | Meaning |
|---|---|
| `Pending Invoice` | Invoice not yet available. Procurement submits the invoice here when it arrives (Path B). Requester uploads supplier invoice here before Procurement processes it (Path C). |

---

## Three Procurement Paths

### Path A — Invoice available immediately
Procurement buys directly and invoice is on hand. Sends Order Confirmation to Requester.
```
Pending Procurement → [Upload Order Confirmation + Submit Invoice]
  → Goods Receipt & Acceptance
  → Pending Accounting → Completed
```

### Path B — Procurement buys directly, invoice arrives late
Procurement buys directly but invoice is not yet available. Sends Order Confirmation to Requester.
GR proceeds first; invoice is submitted at `InvoiceSubmissionScreen` when it arrives.
Accounting always comes last.

```
Pending Procurement → [Upload Order Confirmation + Defer Invoice]
  → Goods Receipt & Acceptance
      │ Accepted, invoice not yet submitted → Pending Invoice (InvoiceSubmissionScreen)
      │ Accepted, invoice already submitted → Pending Accounting
      │ Requires Follow-up → Pending Supplier Follow-up → ... → Pending Invoice → Pending Accounting
  → Pending Invoice (Procurement submits invoice) → Pending Accounting → Completed
```

### Path C — Procurement is intermediary, Requester works directly with Supplier
**Auto-detected** from the original request form — no manual choice by Procurement.
Condition: `ProcurementType = "Invoice Supplied"` AND request has an attachment (Form1).
Requester holds the supplier relationship. No Order Confirmation sent (remittance informs Requester).

```
Pending Procurement → [Upload Remittance + Submit] → email to Requester with remittance
  → Goods Receipt & Acceptance
      │ Requester may attach supplier invoice here (optional)
      │ Accepted → Pending Invoice
      │ Requires Follow-up → Pending Supplier Follow-up (Step 1: Requester)
          │ Requester may attach invoice at Step 1 (optional)
          │ After Step 1 submit → Pending Invoice (unlocks Step 2)
          │ Pending Invoice (Procurement: AI parse + submit) → Pending Supplier Follow-up (Step 2: Procurement enters credit note)
          │ Step 2 submit → Pending Accounting → Completed
      └─ Pending Invoice (Procurement: AI parse + submit) → Pending Accounting → Completed
```

All Path C branches converge at `Pending Accounting → Completed`.
InvoiceSubmissionScreen must complete **before** SFU Step 2.

---

## Routing Logic

### GoodsReceiptScreen — on Accepted

| InvoiceMode | InvoiceSubmitted | Action |
|---|---|---|
| `Direct` | `true` | Status → `Pending Accounting` |
| `Deferred` | `false` | Status → `Pending Invoice` |
| `Deferred` | `true` | Status → `Pending Accounting` |
| `ViaRequester` | — | Status → `Pending Invoice` |

GR "Requires Follow-up": → `Pending Supplier Follow-up` for all InvoiceModes.

### SFU Step 1 (Requester) — on submit, ViaRequester path

Status always → `Pending Invoice` after SFU Step 1 (regardless of whether invoice was attached).
InvoiceSubmissionScreen must complete before SFU Step 2.
- If attachment present at Step 1: patch `RequesterInvoiceURL`, notify Procurement.

### SFU Step 1 (Requester) — on submit, Deferred path

Same routing as GR Accepted (check `InvoiceSubmitted`):
- `InvoiceSubmitted = false` → `Pending Invoice`
- `InvoiceSubmitted = true` → `Pending Accounting`

### Pending Invoice / InvoiceSubmissionScreen — on Procurement submit

| InvoiceMode | SFU Step 2 pending? | Action |
|---|---|---|
| `Deferred` | — | Set `InvoiceSubmitted = true` + invoice data; Status → `Pending Accounting` |
| `ViaRequester` | No (GR Accepted or SFU "accepted") | Set invoice data; Status → `Pending Accounting` |
| `ViaRequester` | Yes (SFU Step 1 = "invoice change", Step 2 not yet done) | Set invoice data; Status → `Pending Supplier Follow-up` (Step 2 now unlocked) |

SFU Step 2 pending = StepNumber 4 log exists AND StepNumber 5 log does not exist.

### SFU Step 2 (Procurement) — on submit

After entering credit note and submitting:
- All InvoiceModes → `Pending Accounting`

### AccountingScreen — on submit

All paths: Status → `Completed`.
No routing logic needed — Accounting is always the final step.

---

## Task Breakdown

### Task 1 — SharePoint Schema
**File to reference:** `docs/sharepoint-schema.md`

Add 5 new columns to `Procurement_Requests` (removed `GRCompleted` and `AccountingCompleted` — no
longer needed since Accounting is always last and there is no parallel merge):

| Column | Type | Values / Notes |
|---|---|---|
| `InvoiceMode` | Choice | `Direct` / `Deferred` / `ViaRequester` — set at Procurement stage |
| `InvoiceSubmitted` | Yes/No | Path B: `true` when Procurement has submitted invoice via InvoiceSubmissionScreen. Default `false`. |
| `OrderConfirmationURL` | Text (single line) | URL to order confirmation image uploaded by Procurement (Path A & B only) |
| `RemittanceURL` | Text (single line) | URL to uploaded remittance file (Path C only) |
| `RequesterInvoiceURL` | Text (single line) | URL to invoice provided by Requester (Path C only) |

---

### Task 2 — ProcurementExecutionScreen
**File:** `Src/ProcurementExecutionScreen.pa.yaml`

**Detect Via Requester path on screen load:**
```
Set(locIsViaRequester,
    gSelectedRequest.ProcurementType.Value = "Invoice Supplied" &&
    !IsBlank(gSelectedRequest.Attachments)
)
```

---

**Branch A/B — Procurement sources the goods (`locIsViaRequester = false`)**

UI changes:
- Add **Order Confirmation upload** control (required before submitting either sub-path).
- Keep the existing 2-way choice:
  - `Submit Invoice Now` → AI parse + submit flow (Path A)
  - `Defer Invoice` → confirm button, no invoice needed (Path B)

Logic — Path A (Direct):
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "Direct",
    OrderConfirmationURL: <uploaded order confirmation URL>,
    Status: "Goods Receipt & Acceptance"
    // + existing invoice fields (InvoiceSubmitted implicitly true for Direct)
})
```
Write `Procurement_ExecutionLog` row (StepNumber 1) with invoice data.
Trigger Flow 7c to notify Requester with order confirmation link.

Logic — Path B (Defer):
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "Deferred",
    OrderConfirmationURL: <uploaded order confirmation URL>,
    Status: "Goods Receipt & Acceptance"
})
```
Write `Procurement_ExecutionLog` row (StepNumber 1) without invoice data.
Trigger Flow 7c to notify Requester with order confirmation link.

---

**Branch C — Requester works directly with Supplier (`locIsViaRequester = true`)**

UI changes:
- Hide Order Confirmation upload and the Submit/Defer choice.
- Show **Remittance upload** control only.

Logic — Path C (Via Requester):
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "ViaRequester",
    RemittanceURL: <uploaded remittance URL>,
    Status: "Goods Receipt & Acceptance"
})
```
Write `Procurement_ExecutionLog` row (StepNumber 1).
Trigger Flow 7a to email Requester with remittance link.

---

### Task 3 — GoodsReceiptScreen
**File:** `Src/GoodsReceiptScreen.pa.yaml`

**UI changes (Path C only):**
- Add an optional invoice attachment section, visible when:
  ```
  gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL)
  ```
- Label: "Attach supplier invoice if you have it — you can also provide it later."
- Not required to complete GR.

**Logic — on submit GR (Accepted):**
- If invoice attachment present (Path C), patch `RequesterInvoiceURL` and trigger Flow 7b.
- Determine next status:
  ```
  If(
      grOutcome = "Requires Follow-up",
      "Pending Supplier Follow-up",
      /* Accepted */
      gSelectedRequest.InvoiceMode = "ViaRequester",
      "Pending Invoice",
      gSelectedRequest.InvoiceMode = "Deferred" && !gSelectedRequest.InvoiceSubmitted,
      "Pending Invoice",
      /* Direct or Deferred with invoice already submitted */
      "Pending Accounting"
  )
  ```

---

### Task 4 — SupplierFollowUpScreen
**File:** `Src/SupplierFollowUpScreen.pa.yaml`

**UI changes (Path C only):**
- Add optional invoice attachment section (same as GoodsReceiptScreen):
  ```
  gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL)
  ```

**Logic — on submit SFU Step 1:**
- If attachment present, patch `RequesterInvoiceURL` and trigger Flow 7b.
- ViaRequester path: always → `Pending Invoice`.
- Deferred path: check `InvoiceSubmitted`:
  - `false` → `Pending Invoice`
  - `true` → `Pending Accounting`

**Logic — on submit SFU Step 2 (Procurement enters credit note):**
SFU Step 2 only runs when Step 1 indicated invoice change. `RequesterInvoiceURL` is guaranteed
set and InvoiceSubmissionScreen has already run. After submit:
- All InvoiceModes → `Pending Accounting`.

---

### Task 5 — Pending Invoice / InvoiceSubmissionScreen (new screen)
**File:** `Src/InvoiceSubmissionScreen.pa.yaml` (new)

**`Pending Invoice` is the status, this screen is what the user sees in that status.**
Also accessible by Procurement from HomeScreen while status is still `Goods Receipt & Acceptance`
or `Pending Supplier Follow-up` (Path B parallel track — badge "Invoice pending" in gallery).

**UI — two sections on the same screen:**

*Requester invoice section* (Path C only, visible when `InvoiceMode = "ViaRequester"`):
- If `RequesterInvoiceURL` blank: file upload + "Attach the supplier invoice when you receive it" + submit.
- If `RequesterInvoiceURL` set: read-only link to file + "Invoice submitted — waiting for Procurement."

*Procurement invoice processing section* (enabled when `InvoiceMode = "Deferred"` OR
`(InvoiceMode = "ViaRequester" && !IsBlank(RequesterInvoiceURL))`):
- Same AI parse flow as ProcurementExecutionScreen (Parse_Invoice + Submit_Invoice flows).
- Shows read-only preview of `RequesterInvoiceURL` file for Path C.
- No defer option — invoice must be submitted here.

**Logic — Requester uploads invoice (Path C):**
```
Patch(Procurement_Requests, gSelectedRequest, { RequesterInvoiceURL: <uploaded URL> })
// Trigger Flow 7b to notify Procurement
// Determine next status:
If(
    // SFU Step 2 still pending
    !IsBlank(LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 4)) &&
    IsBlank(LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 5)),
    "Pending Supplier Follow-up",   // lift SFU Step 2 gate
    "Pending Invoice"               // stay — Procurement will process
)
```

**Logic — Procurement submits invoice (AI parse):**
```
Set(locSFUStep2Pending,
    !IsBlank(LookUp(Procurement_ExecutionLog,
        RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 4)) &&
    IsBlank(LookUp(Procurement_ExecutionLog,
        RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 5))
)

With({wPatched: Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceSubmitted: true,
    Status: If(
        gSelectedRequest.InvoiceMode = "ViaRequester" && locSFUStep2Pending,
        "Pending Supplier Follow-up",   // Step 2 now unlocked
        "Pending Accounting"
    ),
    // + invoice fields
})}, If(IsBlank(wPatched.ID), Notify("Save failed", NotificationType.Error)))
```
Write `Procurement_ExecutionLog` row (StepNumber 6).

---

### Task 5.5 — AccountingScreen
**File:** `Src/AccountingScreen.pa.yaml`

No UI changes needed. Simplify the on-submit routing — Accounting always leads to `Completed`:

```
With({wPatched: Patch(Procurement_Requests, gSelectedRequest, {
    Status: "Completed"
    // + existing accounting fields
})}, If(IsBlank(wPatched.ID), Notify("Save failed", NotificationType.Error)))
```

No InvoiceMode branching needed — `Pending Accounting → Completed` for all paths.

---

### Task 6 — HomeScreen
**File:** `Src/HomeScreen.pa.yaml`

**Changes to Procurement filter:**
- Add `"Pending Invoice"` to the list of statuses Procurement sees in their gallery Items filter.
- Also surface deferred requests where invoice not yet submitted (while status is still GR or SFU):
  ```
  Status in existingProcurementStatuses ||
  (InvoiceMode.Value = "Deferred" && !InvoiceSubmitted &&
   Status in ("Goods Receipt & Acceptance", "Pending Supplier Follow-up"))
  ```
  Show a badge "Invoice pending" on these rows. Tapping navigates to InvoiceSubmissionScreen.

**Changes to Requester filter:**
- Existing `RequesterEmail = gCurrentEmployee.Email` filter covers `Pending Invoice` — no change.
- When Requester taps a `Pending Invoice` request with `InvoiceMode = "ViaRequester"`:
  navigate to InvoiceSubmissionScreen. Upload section visible if `RequesterInvoiceURL` is blank; read-only otherwise.

**Filter button:**
- Add `"Pending Invoice"` as a filter button in the status chip row (visible to Procurement role).

---

### Task 7 — Power Automate Flows

**Flow 7a — Notify Requester: Remittance Ready**
- Trigger: called from ProcurementExecutionScreen (Path C submit)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `remittanceUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with remittance download link and instruction to collect invoice from supplier.

**Flow 7b — Notify Procurement: Invoice Provided by Requester**
- Trigger: called from GoodsReceiptScreen, SupplierFollowUpScreen, or InvoiceSubmissionScreen when `RequesterInvoiceURL` is set
- Parameters: `procurementEmail`, `procurementName`, `requestTitle`, `requestId`, `requesterInvoiceUrl`
- Action: send Outlook email + Teams Adaptive Card to Procurement notifying that Requester has attached the invoice and it is ready to process.

**Flow 7c — Notify Requester: Order Confirmation**
- Trigger: called from ProcurementExecutionScreen on Path A or Path B submit (`locIsViaRequester = false`)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `orderConfirmationUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with the order confirmation image link. Purely informational — no action required from Requester.

> All three flows can be created as standalone flows or consolidated into a single flow
> with a `notificationType` parameter, if parameter sets are compatible.

---

## Implementation Phases (easiest → hardest)

### Phase 0 — Reorder the baseline flow (no schema changes)
**Touch:** `ProcurementExecutionScreen` line 424, `AccountingScreen` line 540, `GoodsReceiptScreen` line 837.

Swap 3 status strings so the correct order `Procurement → GR → Accounting → Completed` is live
for the existing Direct path before any new logic is added:

| File | Change |
|---|---|
| `ProcurementExecutionScreen` | `"Pending Accounting"` → `"Goods Receipt & Acceptance"` |
| `AccountingScreen` | `"Goods Receipt & Acceptance"` → `"Completed"` |
| `GoodsReceiptScreen` | Switch branch `"Accepted" → "Completed"` → `"Accepted" → "Pending Accounting"` |

After this phase the app is functional end-to-end for the simple case. No new columns needed yet.

---

### Phase 1 — SharePoint schema
**Touch:** SharePoint admin UI only — no Power Fx changes.

Add 5 new columns to `Procurement_Requests` (see Task 1). All subsequent phases depend on these
columns existing. `InvoiceSubmitted` defaults to `false`; all URL columns default to blank.

---

### Phase 2 — AccountingScreen (trivial, unblocks testing of Phase 0 + 1)
**Touch:** `Src/AccountingScreen.pa.yaml` — Task 5.5.

Already simplified to `Status → "Completed"` with no branching. Verify the existing submit button
patches this correctly. One-line change if not already done in Phase 0.

---

### Phase 3 — ProcurementExecutionScreen: InvoiceMode + uploads
**Touch:** `Src/ProcurementExecutionScreen.pa.yaml` — Task 2.

Add `locIsViaRequester` detection, Order Confirmation upload (Path A/B), Remittance upload (Path C),
Defer Invoice choice (Path B), and patch `InvoiceMode` + URL columns on submit.
After this phase, every new request gets `InvoiceMode` set correctly.

---

### Phase 4 — GoodsReceiptScreen: routing by InvoiceMode
**Touch:** `Src/GoodsReceiptScreen.pa.yaml` — Task 3.

Update the Accepted branch: `Direct` → `Pending Accounting`, `Deferred` (no invoice) → `Pending Invoice`,
`ViaRequester` → `Pending Invoice`. Add optional invoice attachment section for Path C.
Depends on Phase 1 (columns) and Phase 3 (`InvoiceMode` being set on requests).

---

### Phase 5 — HomeScreen: Pending Invoice filter + badge
**Touch:** `Src/HomeScreen.pa.yaml` — Task 6.

Add `"Pending Invoice"` filter button and gallery Items update for Procurement role.
Add "Invoice pending" badge for Deferred requests still in GR/SFU. Short change, no new screen needed.
Can be done in parallel with Phase 4.

---

### Phase 6 — InvoiceSubmissionScreen: new screen
**Touch:** `Src/InvoiceSubmissionScreen.pa.yaml` (new) — Task 5.

Hardest phase — new screen from scratch. Two sections: Requester invoice upload (Path C) and
Procurement AI parse (Path B/C). Routing on Procurement submit: check `locSFUStep2Pending` to
decide between `Pending Supplier Follow-up` and `Pending Accounting`.
Depends on Phase 1, 3, 4, 5.

---

### Phase 7 — SupplierFollowUpScreen: routing + SFU Step 2 unlock
**Touch:** `Src/SupplierFollowUpScreen.pa.yaml` — Task 4.

Update SFU Step 1 routing (ViaRequester → `Pending Invoice`, Deferred → check `InvoiceSubmitted`).
Add optional invoice attachment section. Wire SFU Step 2 submit to `Pending Accounting`.
Depends on Phase 6 (InvoiceSubmissionScreen must exist before Step 2 unlock logic makes sense).

---

### Phase 8 — Power Automate flows
**Touch:** Power Automate — Task 7. Independent of all Power Fx phases.

Build flows 7a (remittance notify), 7b (invoice provided notify), 7c (order confirmation notify).
Can be started any time after Phase 3 is designed; wire up call sites in Phases 3–6.

---

## Decisions

**Accounting is always the final step.**
`Pending Accounting → Completed` for all paths (Direct, Deferred, ViaRequester). This removes
the need for `GRCompleted` and `AccountingCompleted` merge flags, and eliminates all parallel-track
routing. AccountingScreen no longer needs any InvoiceMode branching.

**Path A: GR now comes before Accounting.**
Previously `Pending Accounting → Goods Receipt`. With Accounting as the final step, Path A becomes
`Goods Receipt & Acceptance → Pending Accounting`. The Direct path sets `InvoiceSubmitted`
implicitly (invoice fields written at ProcurementExecutionScreen), so GR Accepted goes directly
to `Pending Accounting` without passing through `Pending Invoice`.

**Path B parallel track is removed.**
Previously GR and Invoice/Accounting ran in parallel with a merge. Now the flow is strictly
sequential: GR → (if no invoice yet) Pending Invoice → Pending Accounting → Completed.
Procurement can still submit the invoice early via the HomeScreen badge while GR is pending;
doing so sets `InvoiceSubmitted = true` so GR Accepted skips `Pending Invoice` and goes
straight to `Pending Accounting`.

**InvoiceSubmissionScreen StepNumber in `Procurement_ExecutionLog`:** **StepNumber 6**
Distinguishes the deferred/via-requester invoice submission from the initial procurement action (StepNumber 1).
Look up with:
```
LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 6)
```
