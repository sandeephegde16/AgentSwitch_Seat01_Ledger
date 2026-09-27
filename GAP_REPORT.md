# Gap Report — AgentSwitch Ledger Seat vs Campfire

**Seat:** Team 01 · Ledger (`accounting`, `agent`, `crm`)  

## Benchmark

I selected **Campfire** as the primary benchmark because it is an AI-native ERP rather than a traditional accounting product with a chatbot added. Its Ember Agents continuously match bank transactions to GL entries, process AP/AR, detect anomalies and duplicates, manage accruals, and perform scheduled tasks such as flux analysis and close preparation. Agent output is reviewable before posting.

Because this seat represents an Indian company, I also used **Zoho Books India as a compliance reference**, not as the main AI benchmark. Zoho supports IRP e-invoicing, GSTR-1 filing, bank feeds/reconciliation and an official MCP capable of accounting actions.

## 1. What do they do that we do not?

| Gap | Benchmark capability | Current Ledger seat |
|---|---|---|
| **Continuous reconciliation** | Campfire continuously matches bank activity to GL records | Bank transactions exist, but there is no bank-feed ingestion. In the tested data, all 62 bank transactions had `matched_voucher_id = null`; the reconciliation workbench exists on REST but is not exposed to this MCP seat. |
| **Continuous accounting checks** | Campfire detects anomalies, duplicates, miscoding and flux automatically | No accounting anomaly or duplicate-detection agent exists. A read-only scan found negative Cash/Petty Cash, unusually large bank charges, SEZ-tax inconsistencies and bank/receipt linkage problems that nothing had flagged. |
| **Accruals and close automation** | Campfire discovers accruals, proposes journal entries and integrates agent work into the close process | The seat can read POs, bills, periods and locks, but cannot create journal entries or execute close/sign-off. There is no usable close checklist through MCP. |
| **Agent auditability** | Campfire logs agent actions and links results to source accounting data | Audit structures exist, but `/api/audit-log` is inaccessible to this finance seat and `AgentLedgerSeal` contained no records. |
| **India compliance** | Zoho Books supports IRP e-invoicing, GSTR-1 and bank feeds | The platform reports no IRP/IRN generation, no GSTR-1 amendment workflow and no bank-feed connector. These require platform capabilities, not agent reasoning. |

The underlying ledger is nevertheless **internally consistent on the controls tested**: all 11,974 GL entries balance, tested Balance Sheet balances tie to GL sums, and AR ageing reconciles to the AR control account. The major gaps are around workflow, integration, compliance and agent orchestration rather than double-entry integrity.

## 2. Which gaps can an agent close with the tools this seat already has?

A surprising number can be closed today without changing the platform.

**Dispute investigation and credit-note drafting** is the strongest example. An agent can:

`Party → Invoice → PaymentReceived → CreditNote → GL → AccountingPeriod / TransactionLock → CreditNote.create → approval`

It can establish what was billed, what has actually been allocated as payment, previous credits, outstanding exposure, whether the original period is locked, the customer's GST treatment, and then create an invoice-linked **draft** credit note for human approval. The seat already exposes the required credit-note tools and workflow states.

The same tools can also support:

- AR ageing and AR-to-GL reconciliation;
- transaction drill-down from an account balance to source documents;
- anomaly and duplicate scans;
- proposed bank matches;
- close-readiness checks;
- accrual proposals escalated to a human;
- dunning/reminder drafts.

Some apparent gaps are only **MCP surface gaps**. AR/AP ageing, report drill-down, reconciliation workbench and other functions already exist over REST but are not exposed to this seat's MCP. Exposing those capabilities plus date-range filters and aggregation would materially simplify the agent.

An agent **cannot** solve missing bank-feed ingestion, IRP/GSTN integration, GSTR amendment models, journal posting permissions, close/sign-off functionality or inaccessible audit records. These require platform changes.

## 3. What can our agent do that the benchmark product cannot?

The differentiator is not simply “AI accounting”: Campfire already has strong accounting agents. The opportunity is a **platform-native, Indian-accounting agent that combines business context, ledger state and workflow permissions across multiple systems in one goal**.

For example:

> **“A customer disputes last quarter. Work out what we billed, what they paid, and draft the credit note.”**

On the live data, the agent can investigate the selected Vardhman Aerospace SEZ customer across invoices, receipt allocations, bank transactions, GL entries, credit notes, accounting-period locks and GST treatment.

It finds **₹5,04,890 billed, ₹1,71,429.20 allocated as payment and ₹3,33,460.80 outstanding**. For one partially paid invoice, the remaining **₹17,596.80 equals 36 units at the invoiced ₹488.80 rate**, which is useful evidence of a possible quantity dispute—but the agent correctly leaves the reason for a human to confirm.

It can then determine that the original quarter is closed, preserve the customer's SEZ treatment and place of supply, draft the ₹17,596.80 invoice-linked credit note in the open period, explain its arithmetic and assumptions, and submit it for human approval rather than posting autonomously.

Campfire's public material demonstrates sophisticated reconciliation, accrual, anomaly, close and cross-system agents, but I found no public documentation of India-specific GSTN/GSTR/IRN workflows; its published integrations focus instead on banking, payroll, CRM, revenue and sales-tax systems. Zoho provides the India-compliance capabilities, but our opportunity is to combine those jurisdiction-specific accounting rules with this platform's own state machine, CRM context and write-enabled MCP actions.

**Therefore the agent I will build is a dispute investigator and credit-note drafter:** it investigates the full customer/accounting state, identifies ambiguity rather than hiding it, applies period and GST guardrails, creates only a draft, and leaves the financially consequential approval/posting action with a human.