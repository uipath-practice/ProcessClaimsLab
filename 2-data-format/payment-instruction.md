# Payment instruction — Finance Payout Queue

When the Settlement robot processes a claim with `out_FinalDecision ∈ {approve, partial_approve}` (after any reviewer override), it posts a payment instruction to the **Finance Payout Queue** in Orchestrator. Finance Engineering owns the consumer; this lab provides a stub consumer in Dev/Test that drains the queue and logs the messages.

| Aspect | Value |
|---|---|
| Queue name | `Finance Payout Queue` |
| Message format | JSON |
| Producer | Settlement robot |
| Consumer | Finance ERP integration (out of scope; stub in Dev/Test) |
| Retention | Standard Orchestrator queue retention |

---

## 1. Message payload

```jsonc
{
  // ----------- routing -----------
  "claimId":            "CLM-2026-357861",
  "policyId":           "HO-5534416561",

  // ----------- claimant (so finance can populate the beneficiary) -----------
  "claimantName":       "Rajesh Kumar Sharma",
  "claimantEmail":      "rajesh.sharma@example.in",
  "claimantAddress":    {
    "line1":   "12 MG Road, Apt 4B",
    "city":    "Bengaluru",
    "state":   "Karnataka",
    "zip":     "560001",
    "country": "IN"
  },

  // ----------- the money -----------
  "currency":           "INR",                       // ISO 4217
  "netPayoutAmount":    248500.00,                   // bare number; deductible already subtracted
  "settlementBasis":    "replacement_cost",          // "replacement_cost" | "actual_cash_value"

  // ----------- breakdown for reconciliation -----------
  "breakdown": {
    "coverageA":  { "covered_amount": 200000.00, "limit_applied": false, "sublimit_applied": false },
    "coverageB":  { "covered_amount": 0.00,      "limit_applied": false, "sublimit_applied": false },
    "coverageC":  { "covered_amount": 60000.00,  "limit_applied": false, "sublimit_applied": true,
                    "sublimit_categories": [ { "category": "Electronics", "applied": "₹100,000.00" } ] },
    "coverageD":  { "covered_amount": 3500.00,   "limit_applied": false, "sublimit_applied": false },
    "deductible": 15000.00
  },

  // ----------- decision provenance -----------
  "finalDecision":      "partial_approve",                            // approve | partial_approve
  "decisionPath":       "automated",                                  // "automated" | "human_continue" | "human_continue_override"
  "reviewerNotes":      null,                                          // string if a human reviewed; null if automated
  "approvedAt":         "2026-05-27T13:48:00Z",                       // ISO 8601 with tz

  // ----------- letter reference -----------
  "letterReference":    "RESP-CLM-2026-357861-001"                    // out_ClaimResponseJSON.details.letter_metadata.reference
}
```

---

## 2. Field reference

| Field | Type | Required | Description |
|---|---|---|---|
| `claimId` | string | yes | Case key. |
| `policyId` | string | yes | Policy reference. |
| `claimantName` | string | yes | Beneficiary name. |
| `claimantEmail` | string | yes | For payment confirmation correspondence. |
| `claimantAddress` | object | yes | Postal address (for cheque payment fallback or compliance). |
| `currency` | string (ISO 4217) | yes | Policy currency. No FX conversion happens anywhere. |
| `netPayoutAmount` | decimal | yes | Final payable amount after deductible. Always > 0 — `0` payouts do not post to the queue. |
| `settlementBasis` | enum | yes | Mirrors `out_DecisionJSON.details.approved_payout_reference.settlement_basis`. |
| `breakdown` | object | yes | Per-coverage-section detail. Aids finance reconciliation. |
| `finalDecision` | enum: `approve` \| `partial_approve` | yes | Only these two values reach the queue. `deny` does not. |
| `decisionPath` | enum | yes | How the final decision was reached. `automated` = no human in loop. `human_continue` = reviewer continued past escalation. `human_continue_override` = reviewer continued past a denial (force-approve). |
| `reviewerNotes` | string \| null | conditional | Present when `decisionPath` contains "human". |
| `approvedAt` | timestamp | yes | When the final decision was recorded on the case. |
| `letterReference` | string | yes | The decision-letter reference number, useful for cross-referencing in finance systems. |

---

## 3. Producer rules (Settlement robot)

1. **Only post on approve / partial_approve.** Denials do not enqueue a payment instruction.
2. **Zero-payout approvals do not post.** If `netPayoutAmount == 0` (e.g., deductible absorbs the entire loss), the case is settled and closed but no payment instruction is sent — Finance receives nothing because there is nothing to pay.
3. **Settle the basis correctly.** `settlement_basis` comes directly from `out_DecisionJSON.details.approved_payout_reference.settlement_basis`, which is itself set by Agent 04 based on the Replacement Cost Endorsement status in the policy.
4. **Mirror the letter.** `letterReference` matches what the decision letter sent to the claimant carries, so the claimant can quote it back to Finance.
5. **One message per case.** The robot deduplicates: if a message has already been posted for the case (tracked by `paymentInstructionRef` on the case entity), no duplicate is posted on retries.

---

## 4. Consumer expectations (Finance Engineering, future)

Finance Engineering will write a queue consumer that translates these messages into ERP payment runs. The contract above is the API. Specifically:

- `currency` must be supported by the ERP's payable currencies for the relevant insurer entity.
- `decisionPath` allows compliance to differentiate automated and human-reviewed payouts for periodic audit sampling.
- `letterReference` is logged on the ERP transaction memo so a claimant calling Finance can be matched to the case.

---

## 5. Stub consumer behaviour (Dev/Test)

A development stub robot drains the queue and writes the messages to a log table. The stub:

- Acknowledges every message immediately.
- Writes a row to a `FinancePayoutLog` table on the same Data Service tenant.
- Does not call any external payment system.
- Marks the case's `paymentInstructionRef` field with the queue message ID.

---

*End of payment-instruction.md.*
