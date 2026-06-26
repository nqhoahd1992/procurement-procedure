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
| `Pending Invoice` | The InvoiceSubmissionScreen is now active. For Path B: GR completed before Procurement submitted the invoice. For Path C: all branches converge here — Requester uploads supplier invoice and/or Procurement runs AI parse on the same screen. |

> **Key design decision:** `Pending Invoice` is not a waiting room before InvoiceSubmissionScreen.
> `Pending Invoice` IS the InvoiceSubmissionScreen state. The screen handles both Requester invoice
> upload (Path C) and Procurement AI parse in one place. Accessing the screen when status =
> `Goods Receipt & Acceptance` (Path B parallel track) is the same screen opened early.

---

## Three Procurement Paths

### Path A — Invoice available immediately (existing behavior, unchanged)
Procurement buys directly and invoice is on hand. Sends Order Confirmation to Requester.
```
Pending Procurement → [Upload Order Confirmation + Submit Invoice] → Pending Accounting → Goods Receipt → Completed
```

### Path B — Procurement buys directly, invoice arrives late
Procurement buys directly but invoice is not yet available. Sends Order Confirmation to Requester.
Accounting depends only on the official invoice (not on GR), so **invoice→Accounting and GR
run fully in parallel**. Both must complete before the request can reach Completed.

```
Pending Procurement → [Upload Order Confirmation + Defer Invoice]
                                       │
              ┌────────────────────────┴──────────────────────────┐
              │ Track 1: GR (any time)                            │ Track 2: Invoice → Accounting (any time)
              ▼                                                    ▼
   GoodsReceiptScreen                            Pending Invoice / InvoiceSubmissionScreen
   sets GRCompleted = true                       (Procurement opens from HomeScreen badge)
                                                 sets InvoiceSubmitted = true
                                                 → Status: Pending Accounting
                                                 AccountingScreen (Accounting records invoice)
                                                 sets AccountingCompleted = true
              │                                                    │
              └──────────────────── merge ────────────────────────┘
                                        │
                     GRCompleted AND AccountingCompleted → Completed
                     GRCompleted, invoice not yet submitted → Pending Invoice
                     GRCompleted, InvoiceSubmitted, not AccountingCompleted → stay Pending Accounting
                     AccountingCompleted, GR not done → stay Goods Receipt & Acceptance
```

### Path C — Procurement is intermediary, Requester works directly with Supplier
**Auto-detected** from the original request form — no manual choice by Procurement.
Condition: `ProcurementType = "Invoice Supplied"` AND request has an attachment (Form1).
In this path the Requester already holds the supplier relationship, so Order Confirmation is
**not** sent (remittance already informs the Requester that payment was made).

```
Pending Procurement → [Upload remittance + Submit] → email to Requester with remittance
  → Goods Receipt & Acceptance
      │ Requester may attach supplier invoice here if already available (optional)
      ▼
  [GR Accepted] ──────────────────────────────────────────────────────────────────┐
                                                                                   │
  [GR Requires Follow-up] → Pending Supplier Follow-up (SFU Step 1: Requester)   │
      │ Requester may attach invoice at SFU Step 1 (optional)                     │
      │ After Step 1 submit → always → Pending Invoice                            │
      │                                                                            │
      └─────────────────────────────────┐                                          │
                                        ▼                                          ▼
                                  Pending Invoice / InvoiceSubmissionScreen ◄─────┘
                                  ┌─────────────────────────────────────────┐
                                  │ Requester (if RequesterInvoiceURL       │
                                  │ blank): upload supplier invoice         │
                                  │                                         │
                                  │ Procurement (once invoice set):         │
                                  │ AI parse + submit                       │
                                  └─────────────────────────────────────────┘
                                        │ After Procurement submits invoice:
                              ┌─────────┴──────────────────┐
                              │ SFU Step 1 = "invoice       │ SFU Step 1 = "accepted"
                              │ change" → Step 2 pending    │ OR came from GR Accepted
                              ▼                             ▼
              Pending Supplier Follow-up          Pending Accounting
              (SFU Step 2: Procurement             → Completed
               enters credit note)
                              │
                              ▼
                     Pending Accounting → Completed
```

All Path C branches converge at `Pending Invoice`. InvoiceSubmissionScreen must complete
**before** SFU Step 2 — Procurement processes the invoice first, then enters the credit note.
Requester uploads invoice at Pending Invoice if not yet provided at GR or SFU Step 1.

---

## Routing Logic (Path B parallel merge)

### GoodsReceiptScreen / SupplierFollowUpScreen — on Accepted

| InvoiceMode | InvoiceSubmitted | AccountingCompleted | Action |
|---|---|---|---|
| `Direct` | — | — | Status → `Completed` (existing behavior) |
| `Deferred` | `false` | — | Set `GRCompleted = true`; Status → `Pending Invoice` |
| `Deferred` | `true` | `false` | Set `GRCompleted = true`; Status stays `Pending Accounting` |
| `Deferred` | `true` | `true` | Set `GRCompleted = true`; Status → `Completed` |
| `ViaRequester` | — | — | Status → `Pending Invoice` |

GR "Requires Follow-up":
- `Deferred`: Status → `Pending Supplier Follow-up` regardless.
- `ViaRequester`: Status → `Pending Supplier Follow-up` (SFU Step 1 next; invoice gate happens after Step 1).

### SFU Step 1 (Requester) — on submit, ViaRequester path

Status always → `Pending Invoice` after SFU Step 1 (regardless of whether invoice was attached).
InvoiceSubmissionScreen must complete before SFU Step 2.
- If attachment present at Step 1: patch `RequesterInvoiceURL`, notify Procurement.

### SFU Step 2 (Procurement) — on submit, ViaRequester path

SFU Step 2 only occurs when Step 1 selected "invoice change". `RequesterInvoiceURL` is
guaranteed present (InvoiceSubmissionScreen already ran). Procurement enters credit note and submits.
Status → `Pending Accounting`.

### Pending Invoice / InvoiceSubmissionScreen — on Procurement submit

| InvoiceMode | SFU Step 2 pending? | Action |
|---|---|---|
| `Deferred` | — | Set `InvoiceSubmitted = true` + invoice data; Status → `Pending Accounting` |
| `ViaRequester` | No (GR Accepted or SFU "accepted") | Set invoice data; Status → `Pending Accounting` |
| `ViaRequester` | Yes (SFU Step 1 = "invoice change", Step 2 not yet done) | Set invoice data; Status → `Pending Supplier Follow-up` (Step 2 now unlocked) |

SFU Step 2 pending = StepNumber 4 log exists AND StepNumber 5 log does not exist.
For Path B: GR later checks `AccountingCompleted` for the final merge.

### AccountingScreen — on submit (Path B only)

| GRCompleted | Action |
|---|---|
| `false` | Set `AccountingCompleted = true`; Status → `Goods Receipt & Acceptance` (GR still in progress) |
| `true` | Set `AccountingCompleted = true`; Status → `Completed` |

---

## Task Breakdown

### Task 1 — SharePoint Schema
**File to reference:** `docs/sharepoint-schema.md`

Add 7 new columns to `Procurement_Requests`:

| Column | Type | Values / Notes |
|---|---|---|
| `InvoiceMode` | Choice | `Direct` / `Deferred` / `ViaRequester` — set at Procurement stage |
| `InvoiceSubmitted` | Yes/No | Path B: `true` when Procurement has submitted invoice. Default `false`. |
| `GRCompleted` | Yes/No | Path B: `true` when GR (or SFU final step) accepted. Default `false`. Merge flag. |
| `AccountingCompleted` | Yes/No | Path B: `true` when Accounting has finished recording the official invoice. Default `false`. Merge flag. |
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
- Add **Order Confirmation upload** control (required before submitting either sub-path):
  - File upload + label "Upload order confirmation to notify the Requester that the purchase is done."
  - Applies to both Path A and Path B.
- Keep the existing 2-way choice:
  - `Submit Invoice Now` → AI parse + submit flow (Path A)
  - `Defer Invoice` → confirm button, no invoice needed (Path B)

Logic — Path A (Direct):
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "Direct",
    OrderConfirmationURL: <uploaded order confirmation URL>,
    Status: "Pending Accounting"
    // + existing invoice fields
})
```
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
- Hide the Order Confirmation upload section entirely.
- Hide the Submit Invoice Now / Defer Invoice choice.
- Show **Remittance upload** control only:
  - File upload + label "Upload remittance to send to the Requester for supplier invoice collection."

Logic — Path C (Via Requester):
- On upload success, store URL in `RemittanceURL`. On submit:
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
- Not required to complete GR. If left empty, GR submits normally and the request enters `Pending Invoice`.

**Logic — on submit GR (Accepted):**
- If invoice attachment is present (Path C), patch `RequesterInvoiceURL` and trigger Flow 7b.
- Determine next status:
  ```
  If(
      gSelectedRequest.InvoiceMode = "Direct",
      /* existing routing: Completed / Rejected / Pending Supplier Follow-up */,
      grOutcome = "Requires Follow-up",
      "Pending Supplier Follow-up",
      /* Accepted, ViaRequester */
      gSelectedRequest.InvoiceMode = "ViaRequester",
      "Pending Invoice",
      /* Accepted, Deferred — parallel merge */
      !gSelectedRequest.InvoiceSubmitted,
      "Pending Invoice",
      gSelectedRequest.AccountingCompleted,
      "Completed",
      "Pending Accounting"
  )
  ```
- Always patch `GRCompleted: true` alongside the status change (for Deferred path).

---

### Task 4 — SupplierFollowUpScreen
**File:** `Src/SupplierFollowUpScreen.pa.yaml`

**UI changes (Path C only):**
- Add the same optional invoice attachment section:
  ```
  gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL)
  ```
- Not required to submit SFU Step 1.

**Logic — on submit SFU Step 1:**
- If attachment present, patch `RequesterInvoiceURL` and trigger Flow 7b.
- ViaRequester path: always → `Pending Invoice` (InvoiceSubmissionScreen must run before Step 2).
- Deferred path: same routing as GR Accepted (parallel merge table).

**Logic — on submit SFU Step 2 (Procurement enters credit note and submits):**
SFU Step 2 only runs when Step 1 indicated invoice change. `RequesterInvoiceURL` is guaranteed
set and InvoiceSubmissionScreen has already run. After submit:
- `Direct` → `Completed`.
- `ViaRequester` → `Pending Accounting`.
- `Deferred` + not InvoiceSubmitted → `Pending Invoice`.
- `Deferred` + InvoiceSubmitted + AccountingCompleted → `Completed`.
- `Deferred` + InvoiceSubmitted + not AccountingCompleted → stay `Pending Accounting`; set `GRCompleted = true`.

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
// Determine next status
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

No UI changes needed. The only change is the **on-submit routing logic** for Path B (Deferred):

```
If(
    gSelectedRequest.InvoiceMode = "Deferred",
    With({wPatched: Patch(Procurement_Requests, gSelectedRequest, {
        AccountingCompleted: true,
        Status: If(
            gSelectedRequest.GRCompleted,
            "Completed",
            "Goods Receipt & Acceptance"
        )
    })}, If(IsBlank(wPatched.ID), Notify("Save failed", NotificationType.Error))),
    /* Path A / ViaRequester: existing behavior unchanged */
)
```

> When `GRCompleted = false`: status returns to `Goods Receipt & Acceptance` so the GR
> assignee can still access GoodsReceiptScreen via `GRAssignedToID.Id = gCurrentEmployee.ID`.

---

### Task 6 — HomeScreen
**File:** `Src/HomeScreen.pa.yaml`

**Changes to Procurement filter:**
- Add `"Pending Invoice"` to the list of statuses Procurement sees in their gallery Items filter.
- Also surface deferred requests where invoice not yet submitted (parallel track during GR/SFU):
  ```
  Status in existingProcurementStatuses ||
  (InvoiceMode.Value = "Deferred" && !InvoiceSubmitted &&
   Status in ("Goods Receipt & Acceptance", "Pending Supplier Follow-up", "Pending Invoice"))
  ```
  Show a badge "Invoice pending" on these rows. Tapping navigates to InvoiceSubmissionScreen.

**Changes to non-special-role (GR assignee) filter:**
- Verify filter does NOT restrict by status — GR assignee must still see the request when
  status = `Pending Accounting` (Path B parallel, GR not yet done).
- If restricted, extend to include `Pending Accounting` when
  `GRCompleted = false && GRAssignedToID.Id = gCurrentEmployee.ID`.

**Changes to Requester filter:**
- Existing `RequesterEmail = gCurrentEmployee.Email` filter covers `Pending Invoice` — no change.
- When Requester taps a `Pending Invoice` request with `InvoiceMode = "ViaRequester"`:
  navigate to InvoiceSubmissionScreen (= Pending Invoice screen). Upload section visible if
  `RequesterInvoiceURL` is blank; read-only otherwise.

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

## Implementation Order

```
Task 1 (SharePoint) → Task 2 (ProcurementExecutionScreen)
                    → Task 6 (HomeScreen)
                    → Task 3 (GoodsReceiptScreen)
                    → Task 4 (SupplierFollowUpScreen)
                    → Task 5 (InvoiceSubmissionScreen / Pending Invoice screen)
                    → Task 5.5 (AccountingScreen)
                    → Task 7 (Power Automate flows)
```

Tasks 2–6 can proceed after Task 1. Tasks 3 and 4 are independent of each other.
Task 5 and Task 7 can be done in parallel with Tasks 3–4.

---

## Decisions

**HomeScreen: one status per request, not two.**
For Path C Requires Follow-up, Procurement takes two actions sequentially:
(1) `Pending Invoice` → process invoice, then (2) `Pending Supplier Follow-up` → credit note.
The gallery shows whichever status the request is currently in. After Procurement submits at
step (1), the request transitions to step (2) and reappears with the new status. No dual-row display needed.

**InvoiceSubmissionScreen StepNumber in `Procurement_ExecutionLog`:** **StepNumber 6**
Distinguishes the deferred/via-requester invoice submission from the initial procurement action (StepNumber 1).
Look up with:
```
LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 6)
```
