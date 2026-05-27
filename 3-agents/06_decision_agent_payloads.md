### Inputs Prompt
- in_ClaimDataJSON: Cleaned claim data from the 01 Claim Eligibility Agent.
- in_PolicyDataJSON: Structured policy data from the 01 Claim Eligibility Agent.
- in_ClaimEligibilityJSON: Eligibility envelope from Agent 01.
- in_AssessmentValidationJSON: Assessment validation envelope from Agent 02.
- in_CoverageAnalysisJSON: Coverage analysis envelope from Agent 03.
- in_PayoutCalculationJSON: Payout calculation envelope from Agent 04.
- in_CredibilityAssessmentJSON: Credibility assessment envelope from Agent 05.
- in_EscalationDecision_Eligibility: Optional. `"continue"` or `"deny"` when a human reviewer overrode at the eligibility stage. If `"continue"`, do not re-raise the late-filing flag.
- in_EscalationComment_Eligibility: Optional. Free-text reviewer notes.

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
    "in_CredibilityAssessmentJSON"
  ],
  "properties": {
    "in_ClaimDataJSON":              { "type": "object", "description": "Cleaned claim data.", "properties": {} },
    "in_PolicyDataJSON":             { "type": "object", "description": "Structured policy data.", "properties": {} },
    "in_ClaimEligibilityJSON":       { "type": "object", "description": "Eligibility envelope.", "properties": {} },
    "in_AssessmentValidationJSON":   { "type": "object", "description": "Assessment validation envelope.", "properties": {} },
    "in_CoverageAnalysisJSON":       { "type": "object", "description": "Coverage analysis envelope.", "properties": {} },
    "in_PayoutCalculationJSON":      { "type": "object", "description": "Payout calculation envelope.", "properties": {} },
    "in_CredibilityAssessmentJSON":  { "type": "object", "description": "Credibility assessment envelope.", "properties": {} },
    "in_EscalationDecision_Eligibility": {
      "type": "string",
      "description": "Optional. 'continue' or 'deny' when a human reviewer has overridden the eligibility result. If 'continue', do not re-raise the late-filing flag in D-2."
    },
    "in_EscalationComment_Eligibility": {
      "type": "string",
      "description": "Optional. Reviewer's free-text notes."
    }
  },
  "title": "Inputs"
}
```

### Output Format

The Decision Agent produces two outputs.

**`out_DecisionJSON`** — unified envelope. Structure:

```json
{
  "claim_id":       "string",
  "status":         "ok | warn | critical",
  "recommendation": "approve | partial_approve | deny | escalate",
  "checks": [
    { "id": "D-1", "label": "Hard denial gate",       "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "D-2", "label": "Escalation gate",         "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "D-3", "label": "Partial vs full approve", "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "D-4", "label": "Decision priority",       "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] }
  ],
  "findings": [ "string" ],
  "summary":  "string — 2 to 6 sentences",
  "details": {
    "final_decision":                 "approve | partial_approve | deny | escalate",
    "effective_decision_if_continued":"approve | partial_approve | deny",
    "denial_reasons":                 [ "string" ],
    "escalation_reasons":             [ "string" ],
    "approved_payout_reference": {
      "currency":          "string",
      "net_payout":        0.00,
      "settlement_basis":  "replacement_cost | actual_cash_value"
    },
    "confidence_level":               "high | medium | low",
    "escalation_flag_count":          0,
    "decision_priority_applied":      "Deny > Escalate > Partial Approve > Approve"
  }
}
```

**`out_FinalDecision`** — scalar string enum:

```
"approve" | "partial_approve" | "deny" | "escalate"
```

This matches `out_DecisionJSON.details.final_decision`. Maestro routes the case on this scalar.

### Output JSON

```json
{
  "type": "object",
  "required": [
    "out_DecisionJSON",
    "out_FinalDecision"
  ],
  "properties": {
    "out_DecisionJSON": {
      "type": "object",
      "description": "Unified envelope with checks D-1..D-4 plus a details block containing final_decision, effective_decision_if_continued (used by Agent 07 when reviewer continues), denial and escalation reasons, approved payout reference, and confidence level. status is derived strictly from check severities. No absolute monetary thresholds anywhere.",
      "properties": {}
    },
    "out_FinalDecision": {
      "type": "string",
      "description": "Routing scalar matching details.final_decision. One of 'approve', 'partial_approve', 'deny', 'escalate'."
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: India water-damage claim. Coverage full; payout ratio elevated (1.314) → escalation. Credibility medium (estimate ratio is the only flag). Reviewer continued past eligibility late-filing flag.

#### Inputs

```json
{
  "in_ClaimDataJSON":            { "Claim": [ { "ClaimID": "CLM-2026-357861" } ] },
  "in_PolicyDataJSON":           { "coverage_summary": { "currency": "INR" } },
  "in_ClaimEligibilityJSON":     {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "escalate",
    "details":        { "is_eligible_strict": true, "policy_currency": "INR" }
  },
  "in_AssessmentValidationJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "ok",
    "recommendation": "proceed_to_parallel_analysis",
    "details":        { "report_usability": "usable" }
  },
  "in_CoverageAnalysisJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "ok",
    "recommendation": "proceed_to_payout",
    "details":        { "overall_coverage": "full", "exclusions_applied": [], "vacancy_clause": { "applies": false } }
  },
  "in_PayoutCalculationJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "flag_for_review",
    "details": {
      "currency": "INR",
      "reasonableness":               { "claim_total": 335000.00, "assessor_estimate": 255000.00, "ratio": 1.314, "flagged": true },
      "net_payout":                   260000.00,
      "settlement_basis":             "replacement_cost",
      "acv_depreciation_data_missing":false,
      "deductible_absorbs_loss":      false,
      "classification_discrepancies": []
    }
  },
  "in_CredibilityAssessmentJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "escalate_to_human",
    "details": {
      "overall_risk":   "medium",
      "dimension_risk": { "narrative": "low", "estimate": "medium", "documentation": "low", "timing": "low", "prior_claims": "low" },
      "reasonableness": { "claim_total": 335000.00, "assessor_estimate": 255000.00, "ratio": 1.314, "currency": "INR" }
    }
  },
  "in_EscalationDecision_Eligibility": "continue",
  "in_EscalationComment_Eligibility":  "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis."
}
```

#### Outputs

```json
{
  "out_FinalDecision": "escalate",
  "out_DecisionJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "escalate",
    "checks": [
      { "id": "D-1", "label": "Hard denial gate", "result": "pass", "severity": "info",
        "detail": "No hard denial condition present. overall_coverage is 'full' and no cross-currency mismatch is reported.",
        "evidence": [
          { "source": "coverage.details.overall_coverage",                "value": "full" },
          { "source": "payout.details.classification_discrepancies",      "value": "[]" }
        ] },

      { "id": "D-2", "label": "Escalation gate", "result": "fail", "severity": "warn",
        "detail": "Two escalation conditions fired: payout reasonableness ratio 1.314 (>1.20) and credibility overall_risk = medium. Eligibility late-filing was resolved by the human reviewer at the eligibility stage and is not re-raised.",
        "evidence": [
          { "source": "payout.details.reasonableness.ratio",       "value": 1.314 },
          { "source": "credibility.details.overall_risk",          "value": "medium" },
          { "source": "review.eligibility.decision",               "value": "continue" }
        ] },

      { "id": "D-3", "label": "Partial vs full approve", "result": "pass", "severity": "info",
        "detail": "Coverage shape: full. Approve path is 'approve' once escalation is lifted.",
        "evidence": [
          { "source": "coverage.details.overall_coverage", "value": "full" }
        ] },

      { "id": "D-4", "label": "Decision priority", "result": "pass", "severity": "info",
        "detail": "Priority applied: Deny > Escalate > Partial Approve > Approve. D-1 passed; D-2 fired; final decision is 'escalate'.",
        "evidence": [] }
    ],
    "findings": [
      "Final decision: escalate to human reviewer.",
      "Drivers: payout ratio 1.314 (P-8 / CR-2) and credibility overall_risk = medium.",
      "Coverage is full; no items excluded; no Section II exclusion applies.",
      "If the reviewer continues past escalation, the effective decision is 'approve' at ₹260,000 INR (replacement cost basis)."
    ],
    "summary":  "All hard denial gates pass: coverage is full and there are no data errors. Two escalation conditions fired in D-2 — the payout-vs-assessor ratio is 1.314 (above the 1.20 tolerance) and the credibility assessment's overall risk is medium (driven by the same ratio). Per the priority Deny > Escalate > Partial Approve > Approve, the final decision is 'escalate'. The reviewer will see the full analysis and decide whether to continue (effective approve at ₹260,000 INR) or deny. The prior eligibility late-filing concern was already resolved by the reviewer on documented hospitalisation grounds and is not raised here as a separate escalation trigger.",
    "details": {
      "final_decision":                 "escalate",
      "effective_decision_if_continued":"approve",
      "denial_reasons":                 [],
      "escalation_reasons": [
        "payout.details.reasonableness.ratio = 1.314 (>1.20)",
        "credibility.details.overall_risk = medium"
      ],
      "approved_payout_reference": {
        "currency":          "INR",
        "net_payout":        260000.00,
        "settlement_basis":  "replacement_cost"
      },
      "confidence_level":               "medium",
      "escalation_flag_count":          2,
      "decision_priority_applied":      "Deny > Escalate > Partial Approve > Approve"
    }
  }
}
```
