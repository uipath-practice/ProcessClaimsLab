### Inputs Prompt
- in_ClaimDataJSON: Cleaned claim data from the 01 Claim Eligibility Agent.
- in_PolicyDataJSON: Structured policy data from the 01 Claim Eligibility Agent.
- in_AssessmentReportJSON: Structured assessor report from the 02 Assessment Validation Agent.
- in_ClaimEligibilityJSON: Eligibility envelope from Agent 01 — context only.
- in_AssessmentValidationJSON: Assessment validation envelope from Agent 02 — context only.
- in_CoverageAnalysisJSON: Coverage analysis envelope from Agent 03 — use the section_classifications and exclusions_applied as the authoritative coverage decision.
- in_EscalationDecision_Eligibility: Optional. `"continue"` or `"deny"` when a human reviewer overrode at the eligibility stage.
- in_EscalationComment_Eligibility: Optional. Free-text reviewer notes.

### Inputs JSON

```json
{
  "type": "object",
  "required": [
    "in_ClaimDataJSON",
    "in_PolicyDataJSON",
    "in_AssessmentReportJSON",
    "in_ClaimEligibilityJSON",
    "in_AssessmentValidationJSON",
    "in_CoverageAnalysisJSON"
  ],
  "properties": {
    "in_ClaimDataJSON":            { "type": "object", "description": "Cleaned claim data.", "properties": {} },
    "in_PolicyDataJSON":           { "type": "object", "description": "Structured policy data.", "properties": {} },
    "in_AssessmentReportJSON":     { "type": "object", "description": "Structured assessor report.", "properties": {} },
    "in_ClaimEligibilityJSON":     { "type": "object", "description": "Eligibility envelope.", "properties": {} },
    "in_AssessmentValidationJSON": { "type": "object", "description": "Assessment validation envelope.", "properties": {} },
    "in_CoverageAnalysisJSON":     { "type": "object", "description": "Coverage analysis envelope with section_classifications and exclusions_applied — the authoritative coverage decision.", "properties": {} },
    "in_EscalationDecision_Eligibility": {
      "type": "string",
      "description": "Optional. 'continue' or 'deny' when a human reviewer has overridden the eligibility result."
    },
    "in_EscalationComment_Eligibility": {
      "type": "string",
      "description": "Optional. Reviewer's free-text notes. Do not re-raise concerns the reviewer addressed."
    }
  },
  "title": "Inputs"
}
```

### Output Format

The Payout Calculation Agent produces one output: the unified envelope.

**`out_PayoutCalculationJSON`** — structure:

```json
{
  "claim_id":       "string",
  "status":         "ok | warn | critical",
  "recommendation": "approve_payout | flag_for_review | escalate",
  "checks": [
    { "id": "P-1", "label": "Items categorised",                "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "P-2", "label": "Section A limit applied",          "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — include sum and limit", "evidence": [] },
    { "id": "P-3", "label": "Section B limit applied",          "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "P-4", "label": "Section C limit (preliminary)",     "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "P-5", "label": "Section C sublimits",               "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — name capped categories", "evidence": [] },
    { "id": "P-6", "label": "Deductible applied",                "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — include deductible and post-deductible total", "evidence": [] },
    { "id": "P-7", "label": "Settlement basis (RC vs ACV)",      "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — name the endorsement status", "evidence": [] },
    { "id": "P-8", "label": "Reasonableness ratio",              "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — quote claim_total, assessor_estimate, ratio", "evidence": [] },
    { "id": "P-9", "label": "Section D (Loss of Use) limit",     "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] }
  ],
  "findings": [ "string" ],
  "summary":  "string — 2 to 6 sentences; state the final net payout and currency",
  "details": {
    "currency":              "INR",
    "section_totals": {
      "A": { "items_summed": 0, "sum": 0.00, "limit": 0.00, "limit_applied": false, "capped_total": 0.00 },
      "B": { "items_summed": 0, "sum": 0.00, "limit": 0.00, "limit_applied": false, "capped_total": 0.00 },
      "C": { "items_summed": 0, "sum": 0.00, "limit": 0.00, "limit_applied": false, "capped_total": 0.00, "post_sublimits": 0.00 },
      "D": { "items_summed": 0, "sum": 0.00, "limit": 0.00, "limit_applied": false, "capped_total": 0.00 }
    },
    "sublimits_applied":              [ { "category": "string", "category_total": 0.00, "sublimit": 0.00, "amount_capped": 0.00 } ],
    "deductible_applied":             0.00,
    "deductible_absorbs_loss":        false,
    "vacancy_reduction":              { "applies": false, "factor": 0.0 },
    "settlement_basis":               "replacement_cost | actual_cash_value",
    "acv_depreciation_data_missing":  false,
    "reasonableness": {
      "claim_total":      0.00,
      "assessor_estimate": 0.00,
      "ratio":            0.00,
      "flagged":          false
    },
    "net_payout":                     0.00,
    "classification_discrepancies":   []
  }
}
```

### Output JSON

```json
{
  "type": "object",
  "required": [ "out_PayoutCalculationJSON" ],
  "properties": {
    "out_PayoutCalculationJSON": {
      "type": "object",
      "description": "Unified envelope with checks P-1..P-9 plus a details block containing per-section totals, sublimits applied, deductible, settlement basis, reasonableness ratio, and the final net_payout in the policy currency. status is derived strictly from check severities. No absolute monetary thresholds anywhere — escalation is ratio-based and rule-qualitative.",
      "properties": {}
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: same India water-damage claim as the upstream sample. Claim total ₹335,000 vs assessor estimate ₹255,000 → ratio 1.314 (elevated). Coverage A only. RC endorsement included.

#### Inputs

```json
{
  "in_ClaimDataJSON": {
    "Claim": [ { "ClaimID": "CLM-2026-357861" } ],
    "ClaimDamageInventory": [
      { "Category": "Structure - Flooring", "Description": "Engineered timber flooring warped from water exposure, 25 sqm affected", "EstimatedCost": "₹125,000.00", "RepairOrReplace": "Replace" },
      { "Category": "Structure - Drywall",  "Description": "Drywall replacement to 30cm above floor level",                          "EstimatedCost": "₹50,000.00",  "RepairOrReplace": "Replace" },
      { "Category": "Structure - Cabinets", "Description": "Kitchen base cabinet replacement (3 units)",                              "EstimatedCost": "₹100,000.00", "RepairOrReplace": "Replace" }
    ],
    "ClaimClaimTotals": [
      { "TotalStructureDamage": "₹275,000.00", "TotalPersonalProperty": "₹60,000.00", "TotalAdditionalLivingExpenses": "₹0.00", "TotalClaimAmount": "₹335,000.00" }
    ]
  },
  "in_PolicyDataJSON": {
    "coverage_summary": {
      "currency": "INR",
      "coverages": [
        { "coverage_code": "A", "description": "Dwelling",           "limit_of_liability": "₹4,000,000.00", "deductible": "₹15,000.00" },
        { "coverage_code": "B", "description": "Other Structures",   "limit_of_liability": "₹400,000.00",   "deductible": "₹15,000.00" },
        { "coverage_code": "C", "description": "Personal Property",  "limit_of_liability": "₹2,000,000.00", "deductible": "₹15,000.00" },
        { "coverage_code": "D", "description": "Loss of Use",        "limit_of_liability": "₹800,000.00",   "deductible": "None" }
      ],
      "special_limits_of_liability_coverage_c": [
        { "property_category": "Jewelry, watches, furs, and precious stones",   "sublimit": "₹50,000.00" },
        { "property_category": "Electronics, computers, and related equipment", "sublimit": "₹100,000.00" }
      ]
    },
    "section_IV_endorsements": [
      { "endorsement_number": 1, "name": "Replacement Cost Coverage", "status": "Included" }
    ]
  },
  "in_AssessmentReportJSON": {
    "cost_summary": { "total_estimated_repair_cost": "₹255,000.00" }
  },
  "in_ClaimEligibilityJSON":     { "claim_id": "CLM-2026-357861", "recommendation": "escalate" },
  "in_AssessmentValidationJSON": { "claim_id": "CLM-2026-357861", "recommendation": "proceed_to_parallel_analysis" },
  "in_CoverageAnalysisJSON": {
    "claim_id": "CLM-2026-357861",
    "recommendation": "proceed_to_payout",
    "details": {
      "overall_coverage": "full",
      "section_classifications": {
        "A": [ { "item_index": 0, "covered": true }, { "item_index": 1, "covered": true }, { "item_index": 2, "covered": true } ],
        "B": [], "C": [], "D": []
      },
      "exclusions_applied": [],
      "vacancy_clause":     { "applies": false, "reduction": 0.0 }
    }
  },
  "in_EscalationDecision_Eligibility": "continue",
  "in_EscalationComment_Eligibility":  "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis."
}
```

#### Outputs

```json
{
  "out_PayoutCalculationJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "flag_for_review",
    "checks": [
      { "id": "P-1", "label": "Items categorised", "result": "pass", "severity": "info",
        "detail": "All three damage items classified as Coverage A (Dwelling). Agrees with coverage analysis.",
        "evidence": [ { "source": "claim.ClaimDamageInventory[*].Category", "value": "Structure - Flooring, Structure - Drywall, Structure - Cabinets" } ] },

      { "id": "P-2", "label": "Section A limit applied", "result": "pass", "severity": "info",
        "detail": "Sum A = ₹275,000.00; limit = ₹4,000,000.00; not capped.",
        "evidence": [ { "source": "policy.coverage_summary.coverages[code=A].limit_of_liability", "value": "₹4,000,000.00" } ] },

      { "id": "P-3", "label": "Section B limit applied", "result": "n/a", "severity": "info",
        "detail": "No Coverage B items.", "evidence": [] },

      { "id": "P-4", "label": "Section C limit (preliminary)", "result": "n/a", "severity": "info",
        "detail": "No Coverage C items.", "evidence": [] },

      { "id": "P-5", "label": "Section C sublimits", "result": "n/a", "severity": "info",
        "detail": "No Coverage C items; no sublimits applicable.", "evidence": [] },

      { "id": "P-6", "label": "Deductible applied", "result": "pass", "severity": "info",
        "detail": "Deductible ₹15,000.00 subtracted once from A+B+C = ₹275,000.00; post-deductible = ₹260,000.00.",
        "evidence": [ { "source": "policy.coverage_summary.coverages[code=A].deductible", "value": "₹15,000.00" } ] },

      { "id": "P-7", "label": "Settlement basis (RC vs ACV)", "result": "pass", "severity": "info",
        "detail": "Replacement Cost Endorsement is Included. Settlement basis: replacement_cost.",
        "evidence": [ { "source": "policy.section_IV_endorsements[0].status", "value": "Included" } ] },

      { "id": "P-8", "label": "Reasonableness ratio", "result": "pass", "severity": "warn",
        "detail": "Claim total ₹335,000.00 vs assessor estimate ₹255,000.00; ratio 1.314 (elevated). flag_for_review.",
        "evidence": [
          { "source": "claim.ClaimClaimTotals[0].TotalClaimAmount",            "value": "₹335,000.00" },
          { "source": "assessment.cost_summary.total_estimated_repair_cost",   "value": "₹255,000.00" }
        ] },

      { "id": "P-9", "label": "Section D (Loss of Use) limit", "result": "n/a", "severity": "info",
        "detail": "No additional living expenses claimed.", "evidence": [] }
    ],
    "findings": [
      "Net payout: ₹260,000.00 in INR after ₹15,000 deductible.",
      "All claimed items fall under Coverage A (Dwelling); no sublimits engaged.",
      "Settlement at replacement cost — RC endorsement included.",
      "Claim total exceeds the assessor's estimate by 31.4%, above the 20% tolerance. Flagged for review."
    ],
    "summary":  "Net payout is ₹260,000.00 in INR. All three damage items fall under Coverage A (open-peril dwelling). Coverage A sum is ₹275,000 against a ₹4,000,000 limit — well within. The ₹15,000 deductible is subtracted once. Settlement basis is replacement cost (RC endorsement Included). The reasonableness ratio (claim ₹335,000 / assessor ₹255,000 = 1.314) is in the 'elevated' band (1.20–1.50), so the payout is flagged for review per ratio rule P-8. The discrepancy is between the claimant's submission total and the assessor's independent estimate of repair work; the human reviewer can determine whether to settle at the claimant's number, the assessor's, or somewhere between. The prior late-filing acceptance on hospitalisation grounds has no impact on the calculation.",
    "details": {
      "currency": "INR",
      "section_totals": {
        "A": { "items_summed": 3, "sum": 275000.00, "limit": 4000000.00, "limit_applied": false, "capped_total": 275000.00 },
        "B": { "items_summed": 0, "sum": 0.00,      "limit": 400000.00,  "limit_applied": false, "capped_total": 0.00 },
        "C": { "items_summed": 0, "sum": 0.00,      "limit": 2000000.00, "limit_applied": false, "capped_total": 0.00, "post_sublimits": 0.00 },
        "D": { "items_summed": 0, "sum": 0.00,      "limit": 800000.00,  "limit_applied": false, "capped_total": 0.00 }
      },
      "sublimits_applied":             [],
      "deductible_applied":            15000.00,
      "deductible_absorbs_loss":       false,
      "vacancy_reduction":             { "applies": false, "factor": 0.0 },
      "settlement_basis":              "replacement_cost",
      "acv_depreciation_data_missing": false,
      "reasonableness": {
        "claim_total":      335000.00,
        "assessor_estimate": 255000.00,
        "ratio":            1.314,
        "flagged":          true
      },
      "net_payout":                    260000.00,
      "classification_discrepancies":  []
    }
  }
}
```
