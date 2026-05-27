### Inputs Prompt
- in_ClaimDataJSON: Cleaned claim data from the 01 Claim Eligibility Agent.
- in_PolicyDataJSON: Structured policy data from the 01 Claim Eligibility Agent.
- in_ClaimEligibilityJSON: Eligibility envelope from Agent 01.
- in_AssessmentValidationJSON: Assessment validation envelope from Agent 02.
- in_CoverageAnalysisJSON: Coverage analysis envelope from Agent 03.
- in_PayoutCalculationJSON: Payout calculation envelope from Agent 04.
- in_CredibilityAssessmentJSON: Credibility assessment envelope from Agent 05.
- in_DecisionJSON: Decision envelope from Agent 06.
- in_EscalationDecision_Eligibility: Optional. `"continue"` or `"deny"` from eligibility-stage Action Center task.
- in_EscalationComment_Eligibility: Optional. Reviewer's free-text notes at eligibility stage.
- in_EscalationDecision_Decision: Optional. `"continue"` or `"deny"` from decision-stage Action Center task.
- in_EscalationComment_Decision: Optional. Reviewer's free-text notes at decision stage.

### Inputs JSON

```json
{
  "type": "object",
  "required": [
    "in_ClaimDataJSON",
    "in_PolicyDataJSON",
    "in_ClaimEligibilityJSON",
    "in_AssessmentValidationJSON",
    "in_CoverageAnalysisJSON",
    "in_PayoutCalculationJSON",
    "in_CredibilityAssessmentJSON",
    "in_DecisionJSON"
  ],
  "properties": {
    "in_ClaimDataJSON":              { "type": "object", "description": "Cleaned claim data.", "properties": {} },
    "in_PolicyDataJSON":             { "type": "object", "description": "Structured policy data — source for exclusion citations.", "properties": {} },
    "in_ClaimEligibilityJSON":       { "type": "object", "description": "Eligibility envelope.", "properties": {} },
    "in_AssessmentValidationJSON":   { "type": "object", "description": "Assessment validation envelope.", "properties": {} },
    "in_CoverageAnalysisJSON":       { "type": "object", "description": "Coverage analysis envelope — source for section breakdowns and exclusions applied.", "properties": {} },
    "in_PayoutCalculationJSON":      { "type": "object", "description": "Payout calculation envelope — source for amounts, sublimits, deductible, settlement basis.", "properties": {} },
    "in_CredibilityAssessmentJSON":  { "type": "object", "description": "Credibility envelope — context only; never surfaced to the claimant.", "properties": {} },
    "in_DecisionJSON":               { "type": "object", "description": "Decision envelope — source of final_decision and effective_decision_if_continued.", "properties": {} },
    "in_EscalationDecision_Eligibility": { "type": "string", "description": "Optional. 'continue' or 'deny' from the eligibility-stage Action Center task." },
    "in_EscalationComment_Eligibility":  { "type": "string", "description": "Optional. Reviewer's eligibility-stage notes." },
    "in_EscalationDecision_Decision":    { "type": "string", "description": "Optional. 'continue' or 'deny' from the decision-stage Action Center task." },
    "in_EscalationComment_Decision":     { "type": "string", "description": "Optional. Reviewer's decision-stage notes." }
  },
  "title": "Inputs"
}
```

### Output Format

The Claim Response Agent produces one output: the unified envelope. The letter text and its metadata live in the envelope's `details` block (`letter_body` and `letter_metadata`).

**`out_ClaimResponseJSON`** — structure:

```json
{
  "claim_id":       "string",
  "status":         "ok | warn | critical",
  "recommendation": "letter_drafted | letter_pending_review",
  "checks": [
    { "id": "R-1", "label": "Effective decision selected", "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "R-2", "label": "Letter completeness",          "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "R-3", "label": "Reviewer comment incorporated","result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] }
  ],
  "findings": [ "string — internal notes for ops; not for the claimant" ],
  "summary":  "string — internal summary; not for the claimant",
  "details": {
    "effective_decision":  "approve | partial_approve | deny | letter_pending_review",
    "decision_source":     "agent | reviewer_continue | reviewer_deny | pending",
    "letter_metadata": {
      "reference":              "RESP-CLM-2026-357861-001",
      "claim_id":               "CLM-2026-357861",
      "policy_id":              "HO-5534416561",
      "claimant_name":          "string",
      "recipient_email":        "string",
      "currency":               "INR",
      "net_payout_amount":      0.00,
      "settlement_basis":       "replacement_cost | actual_cash_value | null",
      "decision_letter_locale": "en-IN",
      "issued_at":              "2026-05-27T14:30:00Z"
    },
    "letter_body":            "string — the full letter text"
  }
}
```

### Output JSON

```json
{
  "type": "object",
  "required": [ "out_ClaimResponseJSON" ],
  "properties": {
    "out_ClaimResponseJSON": {
      "type": "object",
      "description": "Unified envelope. The letter body and metadata live in details.letter_body and details.letter_metadata. The envelope's summary and findings are internal notes, never surfaced to the claimant. status is derived strictly from check severities.",
      "properties": {}
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: India water-damage claim. Agent 06 escalated due to ratio + credibility. Reviewer at the decision stage **continued** with a note explaining acceptance of the claimant's amount. Effective decision: approve at ₹260,000 INR (replacement cost).

#### Inputs

```json
{
  "in_ClaimDataJSON": {
    "Claim":         [ { "ClaimID": "CLM-2026-357861", "InsurerName": "Apex Mutual Insurance Company", "DateOfSubmission": "26/05/2026" } ],
    "ClaimClaimant": [ { "Name": "Rajesh Kumar Sharma", "PhoneNumber": "+91 98765 43210", "EmailAddress": "rajesh.sharma@example.in", "PolicyNumber": "HO-5534416561" } ],
    "ClaimProperty": [ { "StreetAddress": "12 MG Road, Apt 4B", "City": "Bengaluru", "State": "Karnataka", "ZipCode": "560001", "IsPrimary": true, "IsPresent": true } ]
  },
  "in_PolicyDataJSON": {
    "declarations":     { "policy_number": "HO-5534416561", "named_insured": "Rajesh Kumar Sharma", "producing_agent": { "name": "Anita Iyer", "phone": "+91 80 1234 5678" } },
    "coverage_summary": { "currency": "INR" }
  },
  "in_ClaimEligibilityJSON":     { "claim_id": "CLM-2026-357861", "recommendation": "escalate" },
  "in_AssessmentValidationJSON": { "claim_id": "CLM-2026-357861", "recommendation": "proceed_to_parallel_analysis" },
  "in_CoverageAnalysisJSON":     { "claim_id": "CLM-2026-357861", "recommendation": "proceed_to_payout", "details": { "overall_coverage": "full" } },
  "in_PayoutCalculationJSON": {
    "claim_id": "CLM-2026-357861", "recommendation": "flag_for_review",
    "details": {
      "currency": "INR",
      "section_totals":      { "A": { "capped_total": 275000.00 }, "B": { "capped_total": 0.00 }, "C": { "capped_total": 0.00, "post_sublimits": 0.00 }, "D": { "capped_total": 0.00 } },
      "sublimits_applied":   [],
      "deductible_applied":  15000.00,
      "settlement_basis":    "replacement_cost",
      "net_payout":          260000.00,
      "reasonableness":      { "claim_total": 335000.00, "assessor_estimate": 255000.00, "ratio": 1.314, "flagged": true }
    }
  },
  "in_CredibilityAssessmentJSON": { "claim_id": "CLM-2026-357861", "recommendation": "escalate_to_human", "details": { "overall_risk": "medium" } },
  "in_DecisionJSON": {
    "claim_id":       "CLM-2026-357861",
    "recommendation": "escalate",
    "details": {
      "final_decision":                  "escalate",
      "effective_decision_if_continued": "approve",
      "approved_payout_reference":       { "currency": "INR", "net_payout": 260000.00, "settlement_basis": "replacement_cost" }
    }
  },
  "in_EscalationDecision_Eligibility": "continue",
  "in_EscalationComment_Eligibility":  "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis.",
  "in_EscalationDecision_Decision":    "continue",
  "in_EscalationComment_Decision":     "Ratio of 1.314 acceptable given the assessor's note about local Bengaluru material prices currently running below typical. Approve at the calculated amount."
}
```

#### Outputs

```json
{
  "out_ClaimResponseJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "ok",
    "recommendation": "letter_drafted",
    "checks": [
      { "id": "R-1", "label": "Effective decision selected", "result": "pass", "severity": "info",
        "detail": "Decision-stage reviewer continued; effective_decision_if_continued = 'approve'. Decision source: reviewer_continue.",
        "evidence": [
          { "source": "review.decision.decision",                              "value": "continue" },
          { "source": "decision.details.effective_decision_if_continued",     "value": "approve" }
        ] },

      { "id": "R-2", "label": "Letter completeness", "result": "pass", "severity": "info",
        "detail": "Letter contains header, decision statement, payout amount, breakdown, settlement basis explanation, payment timeline, contact information, and standard closing line.",
        "evidence": [] },

      { "id": "R-3", "label": "Reviewer comment incorporated", "result": "pass", "severity": "info",
        "detail": "Reviewer's acceptance of the claimant's amount is reflected by approving at ₹260,000 (claimant's calculated figure). No explicit mention of the internal review is made in the letter body.",
        "evidence": [
          { "source": "review.decision.comment", "value": "Ratio of 1.314 acceptable given the assessor's note about local Bengaluru material prices currently running below typical. Approve at the calculated amount." }
        ] }
    ],
    "findings": [
      "Effective decision: approve at ₹260,000.00 INR (replacement cost).",
      "Decision source: reviewer_continue (decision-stage Action Center task).",
      "Letter language: English; locale en-IN."
    ],
    "summary":  "Letter drafted for an approved claim of ₹260,000.00 INR. The agent decision was 'escalate' due to elevated payout ratio and medium credibility risk; the decision-stage reviewer continued, accepting the claimant's amount. The letter approves at the calculated net payout, explains the replacement-cost settlement basis, and provides the standard 30-business-day payment timeline. The reviewer's acceptance of the late filing on hospitalisation grounds (eligibility stage) is honoured implicitly by proceeding without re-raising timing.",
    "details": {
      "effective_decision":  "approve",
      "decision_source":     "reviewer_continue",
      "letter_metadata": {
        "reference":              "RESP-CLM-2026-357861-001",
        "claim_id":               "CLM-2026-357861",
        "policy_id":              "HO-5534416561",
        "claimant_name":          "Rajesh Kumar Sharma",
        "recipient_email":        "rajesh.sharma@example.in",
        "currency":               "INR",
        "net_payout_amount":      260000.00,
        "settlement_basis":       "replacement_cost",
        "decision_letter_locale": "en-IN",
        "issued_at":              "2026-05-27T14:30:00Z"
      },
      "letter_body": "Apex Mutual Insurance Company\nClaims Department\nP.O. Box 9400, Hartford, CT 06152\n\n27 May 2026\n\nRef: RESP-CLM-2026-357861-001\nClaim Number: CLM-2026-357861\nPolicy Number: HO-5534416561\n\nMr. Rajesh Kumar Sharma\n12 MG Road, Apt 4B\nBengaluru, Karnataka 560001\nIndia\n\nDear Mr. Sharma,\n\nWe are writing to inform you that your claim CLM-2026-357861 for water damage at 12 MG Road, Apt 4B, Bengaluru has been approved. We have considered the circumstances surrounding the timing of your submission and have proceeded with a full review of your claim.\n\nApproved settlement amount: ₹260,000.00 (Indian Rupees), payable on a replacement cost basis.\n\nBreakdown:\n  • Coverage A (Dwelling) — covered loss subtotal: ₹275,000.00\n  • Coverage B (Other Structures): ₹0.00 (no claim filed)\n  • Coverage C (Personal Property): ₹0.00 (no claim filed)\n  • Coverage D (Loss of Use): ₹0.00 (no claim filed)\n  • Policy deductible applied once: -₹15,000.00\n  • Net payout: ₹260,000.00\n\nBecause your policy includes the Replacement Cost Endorsement, your settlement is calculated at replacement cost without deduction for depreciation, subject to the condition that the property is actually repaired or replaced. Your initial payment will be issued at actual cash value, and the depreciation holdback will be released upon receipt of documentation confirming completion of the repairs (paid invoices, receipts, or contractor sign-off).\n\nYou can expect the initial payment within 30 business days of the date of this letter. The funds will be issued to the bank account on file for your policy; if your account details have changed, please contact our Claims Department to update them before payment is initiated.\n\nIf you have questions about this determination, please contact our Claims Department or your agent at the contact information on your policy declarations page.\n\nSincerely,\n\nClaims Department\nApex Mutual Insurance Company\nclaims@apexmutual.example | 1-800-555-APEX (2739)"
    }
  }
}
```
