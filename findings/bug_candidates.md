# Bug candidates: Ledger seat (Team 01). NOT filed; the human decides.

**Hunt date:** 26 Sep 2026. **Method:** read-only. No write, transition or delete tool was called.

**Seed-data filter applied.** A finding stays only if it's a fault in platform *behaviour* (code, config, metadata, docs) that holds whatever data is in the book, **or** if it reproduces on data that changed *after* seeding. The data was seeded on 12-Sep-2026, and a second system batch on 17-Sep-2026 created 70 draft invoices. Findings visible only in seeded rows were removed (see the end of this file).

Raw API responses and the exploration scripts are not in this repo. Each finding lists the exact call that reproduces it.

---

## HIGH

### 1. Cash Flow statement cannot be produced, on MCP or REST. CONFIRMED
- **Repro (MCP):** `AccountingReport.cash_flow` with:
  - `{from_date:"2026-04-01", to_date:"2026-09-22"}` → **`-32603 Internal server error`**
  - `{from_date:"2025-04-01", to_date:"2026-03-31"}` → `-32603`
  - `{from_date:"2026-09-23", to_date:"2026-09-23"}` → `-32603`
  - `{}` and `{from_date:"2026-04-01", to_date:"2026-09-23"}` → `-32602 "statement requires a single ledger currency; request per-currency reporting"`
- **Repro (REST):** `GET /api/accounting/reports/cash-flow` → 409, same message.
- **Expected:** a cash flow statement, or at worst a clear, actionable error. The company is single-currency: `AccountingReport.trial_balance` returns `mixed_currency: false`, `total_policy: "single-currency"`, `currency_totals: {INR}`.
- **Why it isn't seed data:** a report must not return an internal server error on any data. And the error tells the caller to "request per-currency reporting", but the tool has no currency argument (`from_date, to_date, book_id, view`).
- **Lead:** the only GL rows that carry `currency_code = "INR"` are the two lines of an Expense posted through the app on 23-Sep-2026 (`ab9ccf91…`); all others are null. That fits the 409, but not the 500s.

### 2. Invoices never move to "overdue" (the automatic transition doesn't run). CONFIRMED on post-seed data
- **Definition:** `InvoiceFlow` (`/api/schemas`) has `sent → overdue`, `role: system`, `auto: true`, `condition: "today() > due_date"`.
- **Post-seed evidence:** these 6 invoices were `sent` and **not yet due** when seeded. Their due dates passed between 13 and 23 Sep, and on 26 Sep they are still `sent`. `updated_at` is 2026-09-12 for all of them, so nothing has touched them since:

  | Invoice | Due date |
  |---|---|
  | INV-2026-00151 | 2026-09-13 |
  | INV-2026-00150 | 2026-09-14 |
  | INV-2026-00149 | 2026-09-15 |
  | INV-2026-00148 | 2026-09-17 |
  | INV-2026-00147 | 2026-09-20 |
  | INV-2026-00146 | 2026-09-23 |

- No invoice in the book is in `overdue`.
- **Impact:** anything driven by status (dunning, overdue lists, agent queries such as `Invoice.list(status="overdue")`) silently returns nothing. The ageing report is unaffected because it buckets by date.

---

## MEDIUM

### 3. REST endpoints returning HTTP 500. CONFIRMED (reproduced)
- `GET /api/accounting/indirect-tax/reconcile` → 500, 500, 500
- `GET /api/integrations/jpmorgan/control-totals` → 500, 500, 500
- `GET /api/migration-workbench/batches` → 500, 500, 500
- `GET /api/integrations/datarails/control-totals` → 500, then 200, 200 (intermittent)
- A 500 is a code defect whatever the data. Each should return a result or a 4xx explaining what's missing.

### 4. Safety annotations that mislead MCP clients. CONFIRMED (static metadata)
- 6 tools with `_meta.agentswitch.risk = "DESTRUCTIVE"` have **`annotations.destructiveHint: false`**: `Invoice.cancel.draft.cancelled`, `PaymentReceived.cancel`, `PaymentMade.cancel`, `PurchaseOrder.cancel.draft.cancelled`, `PurchaseRequisition.cancel.draft.cancelled`, `PurchaseRfq.cancel.draft.cancelled`. Clients that gate confirmation on the standard MCP hint won't prompt.
- `endpoint.accounting.bill_match` is described as "Read-only three-way match…" but annotated `readOnlyHint: false`, `risk: WRITE`. An agent under a read-only policy is blocked from a read.
- "Preview" endpoints are labelled as writes: `endpoint.agent_governance.privacy.forget.preview` (WRITE), `endpoint.job_ledger.retention.preview` (POSTING).
- 37 tools have only their HTTP route as the description (e.g. `endpoint.agent_governance.escalations.raise` → "POST /api/agent-governance/escalations/raise"), so agents can't tell what they do.

### 5. Finance seat is given public storefront / booking tools outside its apps. CONFIRMED (static); security review suggested
- `GET /api/auth/me` → `allowed_apps: [accounting, agent, crm]`, yet `tools/list` includes:
  - `endpoint.storefront.checkout` (WRITE)
  - **`endpoint.storefront.payment.verify` (WRITE; args `order_id, order_token, outcome`)**
  - `endpoint.storefront.order.claim`, `endpoint.storefront.notify_me`
  - `endpoint.public.collective.book/waitlist/claim`
  - `endpoint.mission_control.staffing_forecast`
  - `endpoint.job_ledger.verify/replay` (POSTING)
- This contradicts the onboarding rule that the catalogue is scoped to the seat. A payment-verify endpoint that accepts `outcome` from the caller should be reviewed. **Not tested** (write).

---

## LOW

### 6. OpenAPI says "no required parameters", but the server requires them. CONFIRMED
- **422:**
  - `/api/accounting/reports/drill` (account reference)
  - `/api/accounting/flux/consolidated` (`opening_run_id`, `closing_run_id`)
  - `/api/accounting/usage/preview` (`contract_version_id`)
  - `/api/audit-evidence/history` (`entity`, `object_id`), `/api/audit-evidence/trace` (`gl_entry_id` or `voucher_id`)
  - `/api/search`, `/api/portal/kb/search` (`q`)
- **409:** `/api/accounting/ar/customer-hierarchy/leaves` and `/rollup` ("period is required"), `/api/accounting/ar/dunning/trace`.
- **400:** `/api/reports` documents **no parameters at all** but needs `report` ("Available: trial_balance, profit_and_loss, balance_sheet, receivables, payables, stock_balance, gst_r1"); also `/api/finance-workspace/aliases`.

### 7. The `gst_r1` report doesn't state its period. CONFIRMED
- `GET /api/reports?report=gst_r1&from_date=2026-07-01&to_date=2026-07-31` filters correctly (27 invoices, 2–28 Jul) but returns `"period": ""`.

### 8. "Record not found" returned as a JSON-RPC invalid-params error. CONFIRMED (MCP spec nit)
- `<Entity>.get {id: "00000000-0000-0000-0000-000000000000"}` → `-32602` with a message like "Account not found." (all 131 entities with a get tool).
- `-32602` means malformed arguments. MCP expects tool-execution failures in `result.isError`. Agents can only tell "no such record" from "bad arguments" by parsing `data.code`.

### 9. `/metrics` readable by any authenticated user. CONFIRMED
- `GET /metrics` → 200, Prometheus text with 98 metric families: Python and process internals, `auth_failed_total`, email-queue and send-failure counters.
- All `tenant_id` labels are our own tenant, so there's **no cross-tenant leak**. Operational metrics are normally not exposed to end users.

### 10. Trial Balance for a date range omits opening balances. DESIGN?
- `AccountingReport.trial_balance {from_date:"2026-04-01", to_date:<today>}` returns fiscal-year movement only, e.g. 1100 AR −₹76,50,279.26, while the Balance Sheet (which ties to the GL) shows ₹3,55,97,694.74. There's no opening-balance column and no label saying it's movement.
- This fully explains the suspected "CoA vs Balance Sheet" break: the affected accounts are exactly those with pre-April activity (1100, 2100, 1013, 1015 and 1031 have entries before 1-Apr-2026; the matching 1020, 1032, 1014 and 1263 have none). Either label it as movement or include opening balances.

---

## Removed as possibly caused by seeded data
| Old ID | Finding | Why removed |
|---|---|---|
| B2 | Filed GSTR-1 (₹2.49 Cr) ≠ books (₹1.35 Cr) | The `GSTReturn` rows are seeded (round figures, empty `tables`) |
| B4 | Credit notes larger than, or with a different GST treatment from, their invoice | All 25 credit notes were created by the seed (latest 2026-09-12). Missing validation can only be proven by a write |
| B5 | Bank lines "matched" without a voucher; bank vs receipts disconnected | All bank lines and receipts are seeded |
| B6 | Posted invoices have GST only on the header, line heads zero | No invoice has been posted since seeding, so it can't be told apart from seed shape |
| B8 | Impossible GST splits on drafts | The 62 drafts were created in the 17-Sep `system` batch |
| B9 | Negative Cash, Petty Cash, SBI, Advance to Suppliers | Every GL row involved was created by the seed |
| B12 | Draft SEZ invoice charging GST | Created in the 17-Sep `system` batch |
| B13 | PeriodClosing row with negative credit | Closing entries were written at seed time (2026-09-12 17:20:58) |
| B14 | GL `currency_code` / `decimal_places` inconsistency | The difference is exactly seeded rows vs app-posted rows (still the lead for #1) |
| B19 | Payment with bank charges 16.8× the amount | Seeded draft payment |
| B3 (part) | 26 of the 32 past-due "sent" invoices | They were already past due when seeded. Only the 6 that went past due afterwards are kept (#2) |

## Checked and NOT bugs (don't file)
- **5 endpoints rejecting `{}`.** Their schemas mark those fields required, so this is correct.
- **`AcctDocument.list` connection reset.** Transient; it succeeded on 3 retries.
- **403s on other apps and admin routes.** By design.
- **`GET /api/mcp` → 405.** Documented: no SSE.
- **Controls that work:**
  - GL vouchers all balance.
  - Nothing was posted into a closed or locked period after the close or lock.
  - Allocations and outstanding amounts are consistent.
  - All list/get responses match their `outputSchema`.
  - Filters and pagination are correct.
  - Bogus ids fail cleanly.
