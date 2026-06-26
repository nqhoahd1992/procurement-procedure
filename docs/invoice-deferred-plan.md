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
| `Pending Invoice` | GR/SFU completed but invoice not yet submitted. Only reached in Path B when Procurement has not submitted the invoice before GR finishes, or in Path C when Requester has not yet attached the supplier invoice. |

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
   GoodsReceiptScreen                               InvoiceSubmissionScreen (Procurement)
   sets GRCompleted = true                          sets InvoiceSubmitted = true
                                                    → Status: Pending Accounting
                                                    AccountingScreen (Accounting reviews)
                                                    sets AccountingCompleted = true
              │                                                    │
              └──────────────────── merge ────────────────────────┘
                                        │
                     GRCompleted AND AccountingCompleted → Completed
                     GRCompleted, invoice not yet submitted → Pending Invoice (wait for Track 2 start)
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
  [GR Accepted] → Pending Invoice
      │ Requester uploads supplier invoice when it arrives
      │ Once RequesterInvoiceURL is set → notify Procurement → Status stays Pending Invoice
      ▼
  InvoiceSubmissionScreen (Procurement views invoice, AI parse) → Pending Accounting → Completed

  [GR Requires Follow-up] → Pending Supplier Follow-up (SFU Step 1: Requester detail)
      │ Requester may attach supplier invoice at SFU Step 1 if available (optional)
      │ If RequesterInvoiceURL still blank after SFU Step 1 → Status: Pending Invoice
      │   Requester uploads → notify Procurement → Status: back to Pending Supplier Follow-up
      ▼
  SFU Step 2 (Procurement, only when Step 1 indicates invoice change) — enter credit note and submit
      → Pending Accounting → Completed
```

The supplier invoice is **not required at GR or SFU Step 1 time**. `Pending Invoice` acts
as a gate: Procurement cannot proceed (InvoiceSubmissionScreen or SFU Step 2) until
`RequesterInvoiceURL` is set.

---

## Routing Logic (Path B parallel merge)

### GoodsReceiptScreen / SupplierFollowUpScreen — on Accepted

| InvoiceMode | InvoiceSubmitted | AccountingCompleted | Action |
|---|---|---|---|
| `Direct` | — | — | Status → `Completed` (existing behavior) |
| `Deferred` | `false` | — | Set `GRCompleted = true`; Status → `Pending Invoice` (wait for Track 2 to start) |
| `Deferred` | `true` | `false` | Set `GRCompleted = true`; Status stays `Pending Accounting` (Accounting still reviewing) |
| `Deferred` | `true` | `true` | Set `GRCompleted = true`; Status → `Completed` (both tracks done) |
| `ViaRequester` | — | — | Status → `Pending Invoice` (wait for Requester to provide supplier invoice) |

GR "Requires Follow-up":
- `Deferred`: Status → `Pending Supplier Follow-up` regardless.
- `ViaRequester`: Status → `Pending Supplier Follow-up` (SFU Step 1 next; invoice gate happens after Step 1).

### SFU Step 1 (Requester) — on submit, ViaRequester path

| RequesterInvoiceURL | Action |
|---|---|
| Set (attached at SFU Step 1) | Notify Procurement; Status stays `Pending Supplier Follow-up` (Step 2 unlocked if Step 1 indicates invoice change) |
| Blank | Status → `Pending Invoice` (gate: wait for Requester invoice before Step 2) |

### SFU Step 2 (Procurement) — on submit, ViaRequester path

SFU Step 2 only occurs when Step 1 selected "invoice change". Procurement enters credit note
and submits. `RequesterInvoiceURL` is guaranteed present at this point.
Status → `Pending Accounting` directly.

### InvoiceSubmissionScreen — on submit

| GRCompleted | Action |
|---|---|
| `false` | Set `InvoiceSubmitted = true` + invoice data; Status → `Pending Accounting`; Status stays until GR done |
| `true` | Set `InvoiceSubmitted = true` + invoice data; Status → `Pending Accounting` |

Always transitions to `Pending Accounting` — Accounting can start reviewing regardless of GR.

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
| `InvoiceSubmitted` | Yes/No | Path B: `true` when Procurement has submitted invoice via InvoiceSubmissionScreen. Default `false`. |
| `GRCompleted` | Yes/No | Path B: `true` when GR (or SFU final step) accepted. Default `false`. Merge flag. |
| `AccountingCompleted` | Yes/No | Path B: `true` when Accounting has finished receiving/recording the official invoice (AccountingScreen submitted). Default `false`. Merge flag. |
| `OrderConfirmationURL` | Text (single line) | URL to order confirmation image uploaded by Procurement (Path A & B only) |
| `RemittanceURL` | Text (single line) | URL to uploaded remittance file (Path C only) |
| `RequesterInvoiceURL` | Text (single line) | URL to invoice attached by Requester at GR or SFU stage (Path C only) |

---

### Task 2 — ProcurementExecutionScreen
**File:** `Src/ProcurementExecutionScreen.pa.yaml`

**Detect Via Requester path on screen load:**
```
// Set a local variable once when screen opens
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
- Show file upload control → on upload success, store URL in `RemittanceURL`.
- On submit:
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
- If invoice attachment is present (Path C), patch `RequesterInvoiceURL` and trigger notification flow to Procurement.
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
      "Pending Invoice",          // invoice track not started yet
      gSelectedRequest.AccountingCompleted,
      "Completed",                // both tracks done
      /* InvoiceSubmitted but Accounting still reviewing */
      "Pending Accounting"        // status stays on Accounting track; GRCompleted flag set below
  )
  ```
- Always patch `GRCompleted: true` alongside the status change (for Deferred path).

---

### Task 4 — SupplierFollowUpScreen
**File:** `Src/SupplierFollowUpScreen.pa.yaml`

Same pattern as Task 3, applied to Step 1 (Requester detail step):

**UI changes (Path C only):**
- Add the same optional invoice attachment section:
  ```
  gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL)
  ```
- Not required to submit SFU Step 1.

**Logic — on submit SFU Step 1:**
- If attachment present, patch `RequesterInvoiceURL` and trigger notification flow to Procurement.
- Determine next status (ViaRequester path):
  ```
  If(
      gSelectedRequest.InvoiceMode = "ViaRequester" && IsBlank(gSelectedRequest.RequesterInvoiceURL),
      "Pending Invoice",              // invoice gate — Requester must provide before Step 2
      "Pending Supplier Follow-up"    // invoice already set; Step 2 can proceed
  )
  ```

**Logic — on submit SFU Step 2 (Procurement enters credit note and submits):**
SFU Step 2 only runs when Step 1 indicated invoice change. After submit:
- `Direct` → `Completed`.
- `ViaRequester` → `Pending Accounting` (`RequesterInvoiceURL` guaranteed set; Accounting receives invoice next).
- `Deferred` + not InvoiceSubmitted → `Pending Invoice`.
- `Deferred` + InvoiceSubmitted + AccountingCompleted → `Completed`.
- `Deferred` + InvoiceSubmitted + not AccountingCompleted → stay `Pending Accounting`; set `GRCompleted = true`.

---

### Task 5 — InvoiceSubmissionScreen (new screen)
**File:** `Src/InvoiceSubmissionScreen.pa.yaml` (new, based on ProcurementExecutionScreen)

**Purpose:** Procurement submits the official invoice for a deferred request. Can happen
at any time — in parallel with GR (Path B) or after GR is done (both paths).

**Triggered from HomeScreen** for Procurement role when:
```
InvoiceMode = "Deferred" && !InvoiceSubmitted &&
Status in ("Goods Receipt & Acceptance", "Pending Supplier Follow-up", "Pending Invoice")
```
The screen is accessible regardless of whether GR has finished.

**UI:**
- If `InvoiceMode = "ViaRequester"` and `RequesterInvoiceURL` is set:
  show a read-only link/preview so Procurement can view Requester's attached invoice.
- Same AI parse flow as ProcurementExecutionScreen (Parse_Invoice + Submit_Invoice flows).
- No defer option — invoice must be submitted here.

**Logic — on submit:**
```
With({wPatched: Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceSubmitted: true,
    Status: "Pending Accounting",   // always — Accounting starts as soon as invoice is available
    // + invoice fields same as ProcurementExecutionScreen
})}, If(IsBlank(wPatched.ID), Notify("Save failed", NotificationType.Error),
    // success
))
```
Write `Procurement_ExecutionLog` row (StepNumber 6).

> Status always moves to `Pending Accounting` regardless of whether GR is done.
> When GR later completes it checks `AccountingCompleted`: if true → `Completed`;
> if false → stays `Pending Accounting` (Accounting still has control of the status).

---

### Task 5.5 — AccountingScreen
**File:** `Src/AccountingScreen.pa.yaml`

No UI changes needed — Accounting receives and records the official invoice as before.
The only change is the **on-submit routing logic** for Path B (Deferred):

```
If(
    gSelectedRequest.InvoiceMode = "Deferred",
    // Path B: parallel merge — check GR track
    With({wPatched: Patch(Procurement_Requests, gSelectedRequest, {
        AccountingCompleted: true,
        Status: If(
            gSelectedRequest.GRCompleted,
            "Completed",                    // both tracks done
            "Goods Receipt & Acceptance"    // GR still in progress; GR person must still act
        )
    })}, If(IsBlank(wPatched.ID), Notify("Save failed", NotificationType.Error))),
    // Path A / ViaRequester: existing behavior unchanged
    /* existing submit logic */
)
```

> When `GRCompleted = false`: status returns to `Goods Receipt & Acceptance` so the GR
> assignee can still access GoodsReceiptScreen. The request stays visible in their HomeScreen
> filter via `GRAssignedToID.Id = gCurrentEmployee.ID`.

---

### Task 6 — HomeScreen
**File:** `Src/HomeScreen.pa.yaml`

**Changes to Procurement filter:**
- Add `"Pending Invoice"` to the list of statuses Procurement sees in their gallery Items filter.
- Also surface deferred requests where invoice not yet submitted (GR in progress), so
  Procurement can open InvoiceSubmissionScreen in parallel:
  ```
  Status in existingProcurementStatuses ||
  (InvoiceMode.Value = "Deferred" && !InvoiceSubmitted &&
   Status in ("Goods Receipt & Acceptance", "Pending Supplier Follow-up", "Pending Invoice"))
  ```
  Show a badge "Invoice pending" on these rows so Procurement knows they can act now.

**Changes to non-special-role (GR assignee) filter:**
- The existing filter already covers `GRAssignedToID.Id = gCurrentEmployee.ID` across all statuses.
- Verify this filter does NOT restrict by status — the GR assignee must still see the request
  when status = `Pending Accounting` (Accounting running in parallel, GR not yet done).
- If the filter restricts to `Goods Receipt & Acceptance` only, extend it to also include
  `Pending Accounting` when `GRCompleted = false && GRAssignedToID.Id = gCurrentEmployee.ID`.

**Changes to Requester filter:**
- Existing `RequesterEmail = gCurrentEmployee.Email` filter already covers `Pending Invoice` — no change needed.
- When Requester taps a `Pending Invoice` request with `InvoiceMode = "ViaRequester"` and
  `IsBlank(RequesterInvoiceURL)`, navigate to a lightweight **"Provide Supplier Invoice"** view
  (dedicated screen or modal) where Requester uploads the invoice file.
  - On upload: patch `RequesterInvoiceURL`; trigger Flow 7b to notify Procurement.
  - Then determine next status:
    ```
    // Check if SFU Step 1 has been completed (ExecutionLog row StepNumber 4 exists)
    If(
        !IsBlank(LookUp(Procurement_ExecutionLog,
            RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 4)),
        "Pending Supplier Follow-up",   // came from SFU path → unlock Step 2 for Procurement
        "Pending Invoice"               // came from GR path → stay, Procurement uses InvoiceSubmissionScreen
    )
    ```
- When `RequesterInvoiceURL` is already set, the request is read-only for the Requester
  (waiting for Procurement to process it).

**Filter button:**
- Add `"Pending Invoice"` as a filter button in the status chip row (visible to Procurement role).

---

### Task 7 — Power Automate Flows

**Flow 7a — Notify Requester: Remittance Ready**
- Trigger: called from ProcurementExecutionScreen (Path C submit)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `remittanceUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with remittance download link and instruction to collect invoice from supplier.

**Flow 7b — Notify Procurement: Invoice Provided by Requester**
- Trigger: called from GoodsReceiptScreen or SupplierFollowUpScreen when `RequesterInvoiceURL` is set on submit
- Parameters: `procurementEmail`, `procurementName`, `requestTitle`, `requestId`, `requesterInvoiceUrl`
- Action: send Outlook email + Teams Adaptive Card to Procurement notifying that Requester has attached the invoice and it is ready to process.

**Flow 7c — Notify Requester: Order Confirmation**
- Trigger: called from ProcurementExecutionScreen on Path A or Path B submit (i.e. `locIsViaRequester = false`)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `orderConfirmationUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with the order confirmation image link, informing them that Procurement has placed the order. Purely informational — no action required from Requester.

> All three flows can be created as standalone flows or consolidated into a single flow
> with a `notificationType` parameter, if parameter sets are compatible.

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
