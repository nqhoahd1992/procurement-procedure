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
| `Pending Invoice` | Invoice not yet available. Procurement-only screen. Procurement submits the invoice here when it arrives (Path B/C-Deferred). Requester uploads their corrected invoice separately via RequesterInvoiceScreen. |

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
GR proceeds first. Procurement can submit the invoice **at any time** via "Submit Invoice" button
on HomeScreen (visible when `InvoiceMode = "Deferred" && !InvoiceSubmitted`), or wait until
status reaches `Pending Invoice` (set automatically by GR/SFU when invoice is still missing).

```
Pending Procurement → [Upload Order Confirmation + Defer Invoice]
  → Goods Receipt & Acceptance
      ├─ Accepted, InvoiceSubmitted=true  → Pending Accounting → Completed
      │       (Procurement submitted invoice proactively via HomeScreen button during GR)
      ├─ Accepted, InvoiceSubmitted=false → Pending Invoice
      │       → [Procurement submits ISS] → Pending Accounting → Completed
      └─ Requires Follow-up → Pending Supplier Follow-up
              ├─ [SFU Step 1: Accepted] InvoiceSubmitted=true  → Pending Accounting → Completed
              ├─ [SFU Step 1: Accepted] InvoiceSubmitted=false → Pending Invoice
              │       → [Procurement submits ISS] → Pending Accounting → Completed
              ├─ [SFU Step 1: Accepted with Adjustment] InvoiceSubmitted=true
              │       → Pending Supplier Follow-up (Step 2 unlocked)
              │           → [SFU Step 2] → Pending Accounting → Completed
              └─ [SFU Step 1: Accepted with Adjustment] InvoiceSubmitted=false → Pending Invoice
                      → [Procurement submits ISS, locSFUStep2Pending=true]
                          → Pending Supplier Follow-up (Step 2 unlocked)
                              → [SFU Step 2] → Pending Accounting → Completed
```

### Path C — Procurement is intermediary, Requester works directly with Supplier
**Auto-detected** from the original request form — no manual choice by Procurement.
Condition: `!IsBlank(gSelectedRequest.RequesterInvoiceURL)`.
Requester holds the supplier relationship. No Order Confirmation sent (remittance informs Requester).

**At ProcurementExecutionScreen, Procurement must verify the requester's invoice:**

**Path C-Direct** (`rdoInvoiceCorrect_PE = "Yes"`): invoice is correct, submit it now.
```
Pending Procurement → [Verify invoice: Yes] → [AI parse + submit invoice] → [Upload Remittance]
  → Goods Receipt & Acceptance  (InvoiceMode = "ViaRequester")
```

**Path C-Deferred** (`rdoInvoiceCorrect_PE = "No"`): invoice is incorrect, defer.
```
Pending Procurement → [Verify invoice: No] → [Upload Remittance]
  → Goods Receipt & Acceptance  (InvoiceMode = "ViaRequester", InvoiceSubmitted = false)
```

From `Goods Receipt & Acceptance` onward, Path C-Deferred follows the same routing as Path B.
Path C-Direct goes directly to `Pending Accounting` after GR Accepted.

```
                            ┌─── Accepted, InvoiceSubmitted=true → Pending Accounting → Completed
Goods Receipt & Acceptance ─┤
(Path C-Direct always       ├─── Accepted, InvoiceSubmitted=false → Pending Invoice
has InvoiceSubmitted=true   │       → [Procurement submits ISS] → Pending Accounting → Completed
so takes the top branch)    │
                            └─── Requires Follow-up → Pending Supplier Follow-up
                                 ├─ [SFU Step 1: Accepted] InvoiceSubmitted=true
                                 │       → Pending Accounting → Completed
                                 ├─ [SFU Step 1: Accepted] InvoiceSubmitted=false
                                 │       → Pending Invoice → [ISS] → Pending Accounting → Completed
                                 ├─ [SFU Step 1: Accepted with Adjustment] InvoiceSubmitted=true
                                 │       → Pending Supplier Follow-up (Step 2 unlocked)
                                 │           → [SFU Step 2] → Pending Accounting → Completed
                                 └─ [SFU Step 1: Accepted with Adjustment] InvoiceSubmitted=false
                                         → Pending Invoice → [ISS, locSFUStep2Pending=true]
                                             → Pending Supplier Follow-up (Step 2 unlocked)
                                                 → [SFU Step 2] → Pending Accounting → Completed
```

All branches converge at `Pending Accounting → Completed`.
InvoiceSubmissionScreen must complete **before** SFU Step 2 (Path B / Path C-Deferred only).

### Requester invoice upload (Path C-Deferred only)

When Procurement deems the requester's invoice incorrect (`rdoInvoiceCorrect_PE = "No"`),
the Requester can upload the corrected invoice **at any time** via a dedicated
`RequesterInvoiceScreen`, accessible from HomeScreen via a "Submit Invoice" button.

- Button visible when: `InvoiceMode.Value = "ViaRequester" && !InvoiceSubmitted`
- Button disappears once `InvoiceSubmitted = true` (Procurement has processed the invoice)
- Upload simply patches `RequesterInvoiceURL`; triggers Flow 7b to notify Procurement
- No status change — Request continues through its current workflow step

**Detecting Path C-Deferred vs Path B**:
Use `InvoiceMode.Value = "ViaRequester"` — Path C always sets `InvoiceMode = "ViaRequester"`
(regardless of invoice correctness); Path B sets `InvoiceMode = "Deferred"`.

---

## Routing Logic

### GoodsReceiptScreen — on Accepted

Routing depends only on `InvoiceSubmitted` — InvoiceMode does not affect the decision:

| InvoiceSubmitted | Action |
|---|---|
| `true` | Status → `Pending Accounting` |
| `false` | Status → `Pending Invoice` |

GR "Requires Follow-up": → `Pending Supplier Follow-up` for all InvoiceModes.

### SFU Step 1 (Requester) — on submit (all paths)

Routing depends on both **outcome** and **InvoiceSubmitted** — InvoiceMode does not affect the decision:

| Outcome | InvoiceSubmitted | Next Status |
|---|---|---|
| Accepted | `true` | `Pending Accounting` |
| Accepted | `false` | `Pending Invoice` → [ISS] → `Pending Accounting` |
| Accepted with Adjustment | `true` | `Pending Supplier Follow-up` (Step 2 runs immediately) |
| Accepted with Adjustment | `false` | `Pending Invoice` → [ISS, `locSFUStep2Pending=true`] → `Pending Supplier Follow-up` (Step 2 unlocked) |

Requester uploads corrected invoice separately via RequesterInvoiceScreen (accessible from HomeScreen) — not part of the SFU Step 1 form.

### Pending Invoice / InvoiceSubmissionScreen — on Procurement submit

Routing depends only on `locSFUStep2Pending` — InvoiceMode does not affect the decision:

| SFU Step 2 pending? | Action |
|---|---|
| No | Set `InvoiceSubmitted = true` + invoice data; Status → `Pending Accounting` |
| Yes | Set `InvoiceSubmitted = true` + invoice data; Status → `Pending Supplier Follow-up` (Step 2 now unlocked) |

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
| `InvoiceSubmitted` | Yes/No | `true` when Procurement has processed the invoice via `Submit_Invoice.Run` (set at ProcurementExecutionScreen for Path A/C-Direct, at InvoiceSubmissionScreen for Path B/C-Deferred). Default `false`. |
| `OrderConfirmationURL` | Text (single line) | URL to order confirmation image uploaded by Procurement (Path A & B only) |
| `RemittanceURL` | Text (single line) | URL to uploaded remittance file (Path C only) |
| `RequesterInvoiceURL` | Text (single line) | URL to invoice provided by Requester (Path C only) |

---

### Task 2 — ProcurementExecutionScreen
**File:** `Src/ProcurementExecutionScreen.pa.yaml`

**Detect Via Requester path on screen load:**
```
Set(locIsViaRequester, !IsBlank(gSelectedRequest.RequesterInvoiceURL))
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
    InvoiceSubmitted: true,
    OrderConfirmationURL: <uploaded order confirmation URL>,
    Status: "Goods Receipt & Acceptance"
    // + invoice fields
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
- Hide Order Confirmation upload and the Submit/Defer Invoice choice.
- Show invoice verification section: "Is this invoice correct?" (Yes/No).
  - If Yes: show AI parse + invoice fields (same as Path A).
  - If No: hide parse section, checklist items 5&6 also hidden.
- Show **Remittance upload** control (always required).

Logic — Path C-Direct (`rdoInvoiceCorrect_PE = "Yes"`):
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "ViaRequester",
    InvoiceSubmitted: true,
    RemittanceURL: <uploaded remittance URL>,
    OfficialInvoiceLink: gSelectedRequest.RequesterInvoiceURL,
    Status: "Goods Receipt & Acceptance"
})
Submit_Invoice.Run(...)
```
Write `Procurement_ExecutionLog` row (StepNumber 1, "Via Requester").
Trigger Flow 7a to email Requester with remittance link.

Logic — Path C-Deferred (`rdoInvoiceCorrect_PE = "No"`):
```
Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceMode: "ViaRequester",
    InvoiceSubmitted: false,
    RemittanceURL: <uploaded remittance URL>,
    OfficialInvoiceLink: "",
    Status: "Goods Receipt & Acceptance"
})
```
Write `Procurement_ExecutionLog` row (StepNumber 1, "Deferred Invoice").
Trigger Flow 7a to email Requester with remittance link.

---

### Task 3 — GoodsReceiptScreen
**File:** `Src/GoodsReceiptScreen.pa.yaml`

No UI changes needed. Only routing logic update.

**Logic — on submit GR (Accepted):**
- Determine next status:
  ```
  If(
      grOutcome = "Requires Follow-up",
      "Pending Supplier Follow-up",
      /* Accepted — check InvoiceSubmitted (same rule for all InvoiceModes) */
      !gSelectedRequest.InvoiceSubmitted,
      "Pending Invoice",
      "Pending Accounting"
  )
  ```

---

### Task 4 — SupplierFollowUpScreen
**File:** `Src/SupplierFollowUpScreen.pa.yaml`

No UI changes needed. Only routing logic update.

**Logic — on submit SFU Step 1:**
- Routing (same rule for all paths): check `InvoiceSubmitted`:
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
This screen is **Procurement-only** — only reached when status = `Pending Invoice`.
Requester invoice upload is handled separately via `RequesterInvoiceScreen`.

**UI:**

*Requester reminder section* (Path C-Deferred only — visible when `InvoiceMode.Value = "ViaRequester" && IsBlank(RequesterInvoiceURL)`):
- Message: "Waiting for Requester to submit the corrected supplier invoice."
- Button: "Remind Requester" → calls Flow 7d to send Outlook email + Teams Adaptive Card to Requester.
- Procurement invoice processing section is locked (greyed out) until `RequesterInvoiceURL` is set.

*Procurement invoice processing section*:
- For Path B (`InvoiceMode = "Deferred"`): always enabled — Procurement has the invoice.
- For Path C-Deferred (`InvoiceMode = "ViaRequester"`): enabled only after `RequesterInvoiceURL` is set; shows read-only preview of `RequesterInvoiceURL`.
- Same AI parse flow as ProcurementExecutionScreen (Parse_Invoice + Submit_Invoice flows).
- No defer option — invoice must be submitted here.

**Logic — Procurement submits invoice (AI parse):**

`locSFUStep2Pending`: Step 4 log exists AND Step 5 log does not — applies to any InvoiceMode.

Routing depends only on `locSFUStep2Pending` — InvoiceMode does not affect the decision:

| SFU Step 2 pending? | Action |
|---|---|
| No | Status → `Pending Accounting` |
| Yes | Status → `Pending Supplier Follow-up` (Step 2 now unlocked) |

```
Set(locSFUStep2Pending,
    !IsBlank(LookUp(Procurement_ExecutionLog,
        RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 4)) &&
    IsBlank(LookUp(Procurement_ExecutionLog,
        RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 5))
)

With({wPatched: Patch(Procurement_Requests, gSelectedRequest, {
    InvoiceSubmitted: true,
    Status: If(locSFUStep2Pending, "Pending Supplier Follow-up", "Pending Accounting"),
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
- Procurement navigates to InvoiceSubmissionScreen when tapping a `Pending Invoice` request.

**Changes to Requester filter:**
- Existing `RequesterEmail = gCurrentEmployee.Email` filter covers all statuses — no change.
- Add "Submit Invoice" button on gallery row, visible when `InvoiceMode.Value = "ViaRequester" && !InvoiceSubmitted`.
  Tapping navigates to RequesterInvoiceScreen.

**Filter button:**
- Add `"Pending Invoice"` as a filter button in the status chip row (visible to Procurement role).

---

### Task 6.5 — RequesterInvoiceScreen (new screen)
**File:** `Src/RequesterInvoiceScreen.pa.yaml` (new)

Simple single-purpose screen: Requester uploads corrected supplier invoice for Path C-Deferred.

**UI:**
- Read-only header: request title, current status.
- If `RequesterInvoiceURL` already set: read-only link to existing file + "Invoice already submitted — Procurement has been notified."
- File upload form (same Form-based attachment pattern as ProcurementExecutionScreen).
- Submit button.

**Logic — on submit:**
```
// Capture filename before SubmitForm
Set(gPendingNewAttName, Last(attRequesterInvoice.Attachments).Name);
SubmitForm(formRequesterInvoice)

// In formRequesterInvoice.OnSuccess:
With(
    {wURL: gSharePointAttachmentBase & Text(gSelectedRequest.ID) & "/" & EncodeUrl(gPendingNewAttName)},
    With(
        {wReq: Patch(Procurement_Requests, gSelectedRequest, {RequesterInvoiceURL: wURL})},
        If(IsBlank(wReq.ID),
            Notify("Failed to save. Please try again.", NotificationType.Error),
            // Trigger Flow 7b to notify Procurement
            Notify("Invoice submitted. Procurement has been notified.", NotificationType.Success);
            Navigate(HomeScreen)
        )
    )
)
```

No status change. No routing logic.

---

### Task 7 — Power Automate Flows

**Flow 7a — Notify Requester: Remittance Ready**
- Trigger: called from ProcurementExecutionScreen (Path C submit)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `remittanceUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with remittance download link and instruction to collect invoice from supplier.

**Flow 7b — Notify Procurement: Invoice Provided by Requester**
- Trigger: called from RequesterInvoiceScreen when Requester submits their corrected invoice
- Parameters: `procurementEmail`, `procurementName`, `requestTitle`, `requestId`, `requesterInvoiceUrl`
- Action: send Outlook email + Teams Adaptive Card to Procurement notifying that Requester has attached the invoice and it is ready to process.

**Flow 7c — Notify Requester: Order Confirmation**
- Trigger: called from ProcurementExecutionScreen on Path A or Path B submit (`locIsViaRequester = false`)
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`, `orderConfirmationUrl`
- Action: send Outlook email + Teams Adaptive Card to Requester with the order confirmation image link. Purely informational — no action required from Requester.

**Flow 7d — Remind Requester: Invoice Required**
- Trigger: called from InvoiceSubmissionScreen when Procurement clicks "Remind Requester"
- Condition: only reachable when `InvoiceMode = "ViaRequester"` && `IsBlank(RequesterInvoiceURL)`
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`
- Action: send Outlook email + Teams Adaptive Card to Requester asking them to submit the corrected supplier invoice via the app.

> All four flows can be created as standalone flows or consolidated into a single flow
> with a `notificationType` parameter, if parameter sets are compatible.

---

## Implementation Phases (easiest → hardest)

### Phase 0 — Reorder the baseline flow (no schema changes) ✅ DONE

Swapped status strings so the correct order `Procurement → GR → (SFU) → Accounting → Completed`
is live for the existing Direct path. No new columns needed.

| File | Line | Change |
|---|---|---|
| `ProcurementExecutionScreen` | 1424 | `"Pending Accounting"` → `"Goods Receipt & Acceptance"` |
| `AccountingScreen` | 540 | `"Goods Receipt & Acceptance"` → `"Completed"` |
| `GoodsReceiptScreen` | 839 | Switch `"Accepted" → "Completed"` → `"Accepted" → "Pending Accounting"` |
| `GoodsReceiptScreen` | 879 | notify message updated |
| `SupplierFollowUpScreen` | 1463 | Step 1 `"Accepted" → "Completed"` → `"Accepted" → "Pending Accounting"` |
| `SupplierFollowUpScreen` | 1506 | Step 1 notify message updated |
| `SupplierFollowUpScreen` | 1541 | Step 2 `"Completed"` → `"Pending Accounting"` |
| `SupplierFollowUpScreen` | 1572 | Step 2 notify message updated |

---

### Phase 1 — SharePoint schema ✅ DONE

Added 5 columns to `Procurement_Requests`:

| Column | Type |
|---|---|
| `InvoiceMode` | Choice: `Direct` / `Deferred` / `ViaRequester` |
| `InvoiceSubmitted` | Yes/No (default: No) |
| `OrderConfirmationURL` | Single line of text |
| `RemittanceURL` | Single line of text |
| `RequesterInvoiceURL` | Single line of text |

Note: `RequesterInvoiceURL` replaces the previous pattern of reading `gSelectedRequest.Attachments`
as the invoice URL. It is set automatically via post-submit Patch in `RequestFormScreen.Form1.OnSuccess`
(when `ProcurementType = "Invoice Supplied"`). Path C detection uses `!IsBlank(RequesterInvoiceURL)`
instead of `!IsBlank(Attachments)`. The Form1 attachment upload control is kept as-is.

---

### Phase 2 — AccountingScreen ✅ DONE (in Phase 0)

`Status → "Completed"` already fixed in Phase 0 (line 540).

---

### Phase 3 — RequestFormScreen + ProcurementExecutionScreen: InvoiceMode + uploads ✅ DONE
**Touch:** `Src/RequestFormScreen.pa.yaml`, `Src/ProcurementExecutionScreen.pa.yaml` — Task 2.

**RequestFormScreen changes:**
- `Form1.OnSuccess`: add Patch to set `RequesterInvoiceURL` from `Form1.LastSubmit.Attachments.AbsoluteUri`
  when `ProcurementType = "Invoice Supplied"`.

**ProcurementExecutionScreen changes:**
- Replace all `First(gSelectedRequest.Attachments).AbsoluteUri` (5 occurrences) → `gSelectedRequest.RequesterInvoiceURL`.
- Add `locIsViaRequester` on `OnVisible`: `!IsBlank(gSelectedRequest.RequesterInvoiceURL)`.
- Path A/B: add Order Confirmation upload + Submit Invoice Now / Defer Invoice choice.
- Path C: show Remittance upload only; hide invoice section.
- Patch `InvoiceMode` + URL columns on submit.

After this phase, every new request gets `InvoiceMode` set correctly.

---

### Phase 4 — GoodsReceiptScreen: routing by InvoiceMode
**Touch:** `Src/GoodsReceiptScreen.pa.yaml` — Task 3.

Update the Accepted branch: check `InvoiceSubmitted` — `true` → `Pending Accounting`, `false` → `Pending Invoice`.
No UI changes needed.
Depends on Phase 1 (columns) and Phase 3 (`InvoiceMode` being set on requests).

---

### Phase 5 — HomeScreen: Pending Invoice filter + Submit Invoice button
**Touch:** `Src/HomeScreen.pa.yaml` — Task 6.

Add `"Pending Invoice"` filter button and gallery Items update for Procurement role.
Add "Submit Invoice" button on Requester gallery rows (visible when `InvoiceMode.Value = "ViaRequester" && !InvoiceSubmitted`).
Can be done in parallel with Phase 4.

---

### Phase 6 — InvoiceSubmissionScreen + RequesterInvoiceScreen: new screens ✅ DONE
**Touch:** `Src/InvoiceSubmissionScreen.pa.yaml` (new) — Task 5; `Src/RequesterInvoiceScreen.pa.yaml` (new) — Task 6.5.

**InvoiceSubmissionScreen** — Procurement-only. AI parse + submit for Path B/C-Deferred.
- `locSFUStep2Pending` routing: `Pending Supplier Follow-up` or `Pending Accounting`.
- Validations: URL blank, SharePoint domain check, `!gHasInvoiceResult` guard, required fields.
- `rowWaitingForRequester_ISS`: Remind Requester button → calls `Procurement_Notify_Remind_Invoice.Run(...)`.
- `rowRequesterInvoicePreview_ISS`: clickable link when `ViaRequester` + URL set.

**RequesterInvoiceScreen** — Requester-only. Upload + patch `RequesterInvoiceURL`.
- Re-upload allowed at any time while `InvoiceSubmitted = false`.
- Remove invoice: clears `RequesterInvoiceURL` + refreshes `gSelectedRequest`.
- `InvoiceSubmitted = true`: screen locked (banner only, no upload section).
- On success: calls `Procurement_Notify_Invoice_Provided.Run(title, requestId, url)`.

**HomeScreen navigation wiring** (done alongside this phase):
- Gallery `OnSelect` (all 6 hit-areas): routes to `InvoiceSubmissionScreen` when
  `Status = "Pending Invoice" && Role = "Procurement" || "Admin"`; otherwise `RequestDetailScreen`.

---

### Phase 7 — SupplierFollowUpScreen: routing ✅ DONE
**Touch:** `Src/SupplierFollowUpScreen.pa.yaml` — Task 4.

- `btnSubmitStep1_SFU`: `Accepted` → `If(!wLatest.InvoiceSubmitted, "Pending Invoice", "Pending Accounting")`.
- `btnSubmitStep2_SFU`: → `If(!gSelectedRequest.InvoiceSubmitted, "Pending Invoice", "Pending Accounting")`.
- Notify messages updated to reflect dynamic routing.

---

### Phase 8 — Power Automate flows
**Touch:** Power Automate — Task 7. Canvas App call sites already wired.

**Canvas App side — already done:**
| Flow | Call site | Parameters |
|---|---|---|
| 7b `Procurement_Notify_Invoice_Provided` | `RequesterInvoiceScreen.formRequesterInvoice.OnSuccess` | `title`, `requestId`, `requesterInvoiceUrl` |
| 7d `Procurement_Notify_Remind_Invoice` | `InvoiceSubmissionScreen.btnRemindRequester_ISS.OnSelect` | `requesterEmail`, `requesterName`, `title`, `requestId` |
| 7a + 7c | Triggered automatically via `Procurement_ExecutionLog_Created` (StepNumber 1) | No Canvas App changes needed |

**Power Automate side — to do:**

**Flow 7a + 7c** — integrate into existing `Procurement_ExecutionLog_Created` flow, case `StepNumber = 1`:
- Add condition on `InvoiceMode`:
  - `ViaRequester` → 7a: email + Teams card to Requester with `RemittanceURL`.
  - `Direct` / `Deferred` → 7c: email + Teams card to Requester with `OrderConfirmationURL`.

**Flow 7b** — new instant flow `Procurement_Notify_Invoice_Provided`:
- Parameters: `requestTitle` (text), `requestId` (number), `requesterInvoiceUrl` (text).
- Lookup `Procurement_ExecutionLog` where `RequestIDText = requestId && StepNumber = 1` → get `ExecutedBy` email.
- Send email + Teams Adaptive Card to that Procurement person.

**Flow 7d** — new instant flow `Procurement_Notify_Remind_Invoice`:
- Parameters: `requesterEmail`, `requesterName`, `requestTitle`, `requestId`.
- Send email + Teams Adaptive Card to Requester asking them to submit corrected invoice via app.

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
InvoiceSubmissionScreen is only accessible when status = `Pending Invoice`.

**InvoiceSubmissionScreen StepNumber in `Procurement_ExecutionLog`:** **StepNumber 6**
Distinguishes the deferred/via-requester invoice submission from the initial procurement action (StepNumber 1).
Look up with:
```
LookUp(Procurement_ExecutionLog, RequestIDText = Text(gSelectedRequest.ID) && StepNumber = 6)
```
