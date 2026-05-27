### Inputs Prompt
- in_ClaimDataJSON: Cleaned claim data from the 01 Claim Eligibility Agent.
- in_PolicyDataJSON: Structured policy data from the 01 Claim Eligibility Agent.
- in_AssessmentReportJSON: Structured assessor report from the 02 Assessment Validation Agent.
- in_ClaimEligibilityJSON: Eligibility envelope from Agent 01 — context only.
- in_AssessmentValidationJSON: Assessment validation envelope from Agent 02 — context only.
- in_EscalationDecision_Eligibility: Optional. `"continue"` or `"deny"` when a human reviewer overrode at the eligibility stage. Honour the human decision.
- in_EscalationComment_Eligibility: Optional. Free-text reviewer notes. Do not re-raise concerns the reviewer has addressed.

### Inputs JSON

```json
{
  "type": "object",
  "required": [
    "in_ClaimDataJSON",
    "in_PolicyDataJSON",
    "in_AssessmentReportJSON",
    "in_ClaimEligibilityJSON",
    "in_AssessmentValidationJSON"
  ],
  "properties": {
    "in_ClaimDataJSON":              { "type": "object", "description": "Cleaned claim data.", "properties": {} },
    "in_PolicyDataJSON":             { "type": "object", "description": "Structured policy data.", "properties": {} },
    "in_AssessmentReportJSON":       { "type": "object", "description": "Structured assessor report.", "properties": {} },
    "in_ClaimEligibilityJSON":       { "type": "object", "description": "Eligibility envelope, context only.", "properties": {} },
    "in_AssessmentValidationJSON":   { "type": "object", "description": "Assessment validation envelope, context only.", "properties": {} },
    "in_EscalationDecision_Eligibility": {
      "type": "string",
      "description": "Optional. 'continue' or 'deny' when a human reviewer has overridden the eligibility result. Honour the human decision."
    },
    "in_EscalationComment_Eligibility": {
      "type": "string",
      "description": "Optional. Reviewer's free-text notes explaining the eligibility override. Do not re-raise concerns the reviewer addressed."
    }
  },
  "title": "Inputs"
}
```

### Output Format

The Coverage Analysis Agent produces one output: the unified envelope.

**`out_CoverageAnalysisJSON`** — structure:

```json
{
  "claim_id":       "string",
  "status":         "ok | warn | critical",
  "recommendation": "proceed_to_payout | deny_all | partial_approval | escalate",
  "checks": [
    {
      "id":       "C-1",
      "label":    "Peril identified",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "claim.ClaimIncident[0].TypeOfIncident",   "value": "string" },
        { "source": "assessment.findings.cause_determination", "value": "string" }
      ]
    },
    {
      "id":       "C-2",
      "label":    "Section II exclusions",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string — cite exclusion number+name if any apply",
      "evidence": [
        { "source": "policy.section_II_exclusions.A_excluded_perils", "value": "<exclusion considered>" }
      ]
    },
    { "id": "C-3", "label": "Coverage A applies (Dwelling)",          "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "C-4", "label": "Coverage B applies (Other Structures)",  "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "C-5", "label": "Coverage C applies (Personal Property)", "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — name the matching named peril if applicable", "evidence": [] },
    { "id": "C-6", "label": "Coverage D applies (Loss of Use)",       "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "C-7", "label": "Vacancy clause",                          "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "C-8", "label": "Duty to mitigate",                        "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] }
  ],
  "findings": [ "string" ],
  "summary":  "string — 2 to 6 sentences",
  "details": {
    "working_peril":           "string",
    "peril_source":            "claimant | assessor | reconciled",
    "overall_coverage":        "full | partial | none",
    "section_classifications": {
      "A": [ { "item_index": 0, "covered": true,  "rationale": "string" } ],
      "B": [],
      "C": [ { "item_index": 0, "covered": true,  "matched_peril": "string", "rationale": "string" } ],
      "D": [ { "item_index": 0, "covered": true,  "rationale": "string" } ]
    },
    "exclusions_applied":      [ { "exclusion_number": 0, "exclusion_name": "string", "applies_to": "A | B | C | D" } ],
    "vacancy_clause":          { "applies": false, "reduction": 0.0 },
    "duty_to_mitigate":        { "evidence_present": true, "deduction_recommended": false },
    "ambiguities":             [ "string" ]
  }
}
```

### Output JSON

```json
{
  "type": "object",
  "required": [ "out_CoverageAnalysisJSON" ],
  "properties": {
    "out_CoverageAnalysisJSON": {
      "type": "object",
      "description": "Unified envelope with checks C-1..C-8 plus a details block describing the working peril, per-item classifications by coverage section, exclusions applied, vacancy-clause findings, and mitigation evidence. status is derived strictly from check severities.",
      "properties": {}
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: synthetic-document generator, India locale (INR). Burst-pipe water-damage claim — a typical "covered under Coverage A & C12" outcome.

#### Inputs

(Same `in_ClaimDataJSON`, `in_PolicyDataJSON`, and `in_AssessmentReportJSON` shape as the upstream agents emit. Reviewer continued past the eligibility late-filing flag.)

```json
{
  "in_ClaimDataJSON":            { "Claim": [ { "ClaimID": "CLM-2026-357861" } ], "ClaimIncident": [ { "TypeOfIncident": "Water Damage", "TemporaryRepairsMade": true, "TemporaryRepairsDescription": "Main water shutoff closed; affected area dried with fans for 24h." } ], "ClaimDamageInventory": [ { "Category": "Structure - Flooring", "Description": "Engineered timber flooring warped from water exposure, 25 sqm affected", "EstimatedCost": "₹125,000.00", "RepairOrReplace": "Replace" } ], "ClaimClaimTotals": [ { "TotalAdditionalLivingExpenses": "₹0.00" } ], "ClaimProperty": [ { "IsPrimary": true, "IsPresent": true } ] },
  "in_PolicyDataJSON":           { "coverage_summary": { "currency": "INR" }, "section_I_covered_perils": { "coverage_C": { "perils": [ { "number": 12, "name": "Accidental Discharge or Overflow of Water or Steam" } ] } }, "section_II_exclusions": { "A_excluded_perils": [ { "number": 1, "name": "Flood" }, { "number": 3, "name": "Water Damage" } ] }, "section_III_special_conditions": [ { "number": 4, "name": "Vacancy Clause", "value": "60 consecutive days; 15% reduction" } ] },
  "in_AssessmentReportJSON":     { "findings": { "cause_determination": "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident." } },
  "in_ClaimEligibilityJSON":     { "claim_id": "CLM-2026-357861", "recommendation": "escalate" },
  "in_AssessmentValidationJSON": { "claim_id": "CLM-2026-357861", "recommendation": "proceed_to_parallel_analysis" },
  "in_EscalationDecision_Eligibility": "continue",
  "in_EscalationComment_Eligibility":  "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis."
}
```

#### Outputs

```json
{
  "out_CoverageAnalysisJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "ok",
    "recommendation": "proceed_to_payout",
    "checks": [
      { "id": "C-1", "label": "Peril identified", "result": "pass", "severity": "info",
        "detail": "Working peril is Accidental Discharge or Overflow of Water or Steam (interior plumbing failure). Claimant reports 'Water Damage'; assessor confirms burst copper feeder pipe.",
        "evidence": [
          { "source": "claim.ClaimIncident[0].TypeOfIncident",   "value": "Water Damage" },
          { "source": "assessment.findings.cause_determination", "value": "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident." }
        ] },

      { "id": "C-2", "label": "Section II exclusions", "result": "pass", "severity": "info",
        "detail": "Section II A-1 (Flood) does not apply: this is interior plumbing failure, not water from outside the building. Section II A-3 (Water Damage) does not apply: the exclusion targets sewer/drain backup, surface water, and below-grade seepage — not sudden interior plumbing failure, which is named-peril 12 under Coverage C and falls under open-peril coverage for Coverage A.",
        "evidence": [
          { "source": "policy.section_II_exclusions.A_excluded_perils[0]", "value": "1. Flood" },
          { "source": "policy.section_II_exclusions.A_excluded_perils[2]", "value": "3. Water Damage" }
        ] },

      { "id": "C-3", "label": "Coverage A applies (Dwelling)", "result": "pass", "severity": "info",
        "detail": "Engineered timber flooring damage is structural damage to the dwelling. Coverage A is open-peril; no Section II exclusion applies; covered.",
        "evidence": [
          { "source": "claim.ClaimDamageInventory[0]", "value": "Structure - Flooring: Engineered timber flooring warped from water exposure" }
        ] },

      { "id": "C-4", "label": "Coverage B applies (Other Structures)", "result": "pass", "severity": "info",
        "detail": "No Coverage B items claimed (no damage to detached structures).",
        "evidence": [] },

      { "id": "C-5", "label": "Coverage C applies (Personal Property)", "result": "pass", "severity": "info",
        "detail": "No personal-property items in this claim's inventory. Would otherwise match named peril 12: Accidental Discharge or Overflow of Water or Steam.",
        "evidence": [
          { "source": "policy.section_I_covered_perils.coverage_C.perils[11]", "value": "12. Accidental Discharge or Overflow of Water or Steam" }
        ] },

      { "id": "C-6", "label": "Coverage D applies (Loss of Use)", "result": "pass", "severity": "info",
        "detail": "TotalAdditionalLivingExpenses is ₹0; no Coverage D claim.",
        "evidence": [
          { "source": "claim.ClaimClaimTotals[0].TotalAdditionalLivingExpenses", "value": "₹0.00" }
        ] },

      { "id": "C-7", "label": "Vacancy clause", "result": "pass", "severity": "info",
        "detail": "Property was owner-occupied at the time of loss (IsPrimary=true, IsPresent=true). Vacancy clause does not apply.",
        "evidence": [
          { "source": "claim.ClaimProperty[0].IsPrimary", "value": "true" },
          { "source": "claim.ClaimProperty[0].IsPresent", "value": "true" }
        ] },

      { "id": "C-8", "label": "Duty to mitigate", "result": "pass", "severity": "info",
        "detail": "Claimant closed the main water shutoff and dried the affected area for 24h. Mitigation evidence is present.",
        "evidence": [
          { "source": "claim.ClaimIncident[0].TemporaryRepairsMade",        "value": "true" },
          { "source": "claim.ClaimIncident[0].TemporaryRepairsDescription", "value": "Main water shutoff closed; affected area dried with fans for 24h." }
        ] }
    ],
    "findings": [
      "Loss is from an interior plumbing failure — a covered peril.",
      "No Section II exclusion applies. The 'Water Damage' exclusion (A-3) targets sewer backup and surface water, not interior plumbing failure.",
      "Coverage A (Dwelling) applies to the timber-flooring damage.",
      "Claimant fulfilled the duty to mitigate (shutoff closed, drying initiated)."
    ],
    "summary":  "The loss is a covered peril under this HO-3 policy. The assessor confirms a burst copper feeder pipe — interior plumbing failure — which falls under Coverage A (open-peril dwelling coverage) for the structural damage and would fall under named-peril 12 for any personal-property damage. Section II exclusions A-1 (Flood) and A-3 (Water Damage) do not apply: the former requires water from a body of water or surface water; the latter targets sewage/drain backup and below-grade seepage. The vacancy clause is moot — the property was owner-occupied at the time of loss. Mitigation evidence is present. The claim proceeds to payout calculation. The human reviewer's prior acceptance of the late filing is noted; no coverage issue arises from that decision.",
    "details": {
      "working_peril":           "Accidental Discharge or Overflow of Water or Steam (interior plumbing failure)",
      "peril_source":            "reconciled",
      "overall_coverage":        "full",
      "section_classifications": {
        "A": [ { "item_index": 0, "covered": true, "rationale": "Structural timber-flooring damage from covered peril; open-peril Coverage A applies." } ],
        "B": [],
        "C": [],
        "D": []
      },
      "exclusions_applied":      [],
      "vacancy_clause":          { "applies": false, "reduction": 0.0 },
      "duty_to_mitigate":        { "evidence_present": true, "deduction_recommended": false },
      "ambiguities":             []
    }
  }
}
```
