### Inputs Prompt
- in_ClaimDataJSON: Cleaned claim data from the 01 Claim Eligibility Agent.
- in_PolicyDataJSON: Structured policy data from the 01 Claim Eligibility Agent.
- in_AssessmentReportJSON: Structured assessor report from the 02 Assessment Validation Agent.
- in_ClaimEligibilityJSON: Eligibility envelope from Agent 01 — context only.
- in_AssessmentValidationJSON: Assessment validation envelope from Agent 02 — context only.
- in_PriorClaimsJSON: Array of prior claims for the same claimant or policy. May be an empty array if no priors are known.
- in_EscalationDecision_Eligibility: Optional. `"continue"` or `"deny"` when a human reviewer overrode at the eligibility stage. If `continue` AND the comment explains timing, downgrade CR-4.
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
    "in_PriorClaimsJSON"
  ],
  "properties": {
    "in_ClaimDataJSON":            { "type": "object", "description": "Cleaned claim data.", "properties": {} },
    "in_PolicyDataJSON":           { "type": "object", "description": "Structured policy data.", "properties": {} },
    "in_AssessmentReportJSON":     { "type": "object", "description": "Structured assessor report.", "properties": {} },
    "in_ClaimEligibilityJSON":     { "type": "object", "description": "Eligibility envelope.", "properties": {} },
    "in_AssessmentValidationJSON": { "type": "object", "description": "Assessment validation envelope.", "properties": {} },
    "in_PriorClaimsJSON": {
      "type": "array",
      "description": "Array of prior claims for the same claimant or policy. Each element has claimId, incidentDate, incidentType, totalClaimAmount, currency, finalDecision, closedAt.",
      "items": { "type": "object", "properties": {} }
    },
    "in_EscalationDecision_Eligibility": {
      "type": "string",
      "description": "Optional. 'continue' or 'deny' when a human reviewer overrode the eligibility result. If 'continue' and the comment explains timing, downgrade CR-4."
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

The Credibility Assessment Agent produces one output: the unified envelope.

**`out_CredibilityAssessmentJSON`** — structure:

```json
{
  "claim_id":       "string",
  "status":         "ok | warn | critical",
  "recommendation": "proceed_to_decision | escalate_to_human",
  "checks": [
    { "id": "CR-1", "label": "Narrative consistency",      "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — include the per-dimension risk: low | medium | high", "evidence": [] },
    { "id": "CR-2", "label": "Estimate reasonableness",    "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — include ratio and per-dimension risk", "evidence": [] },
    { "id": "CR-3", "label": "Documentation completeness", "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] },
    { "id": "CR-4", "label": "Timing and pattern",         "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string — include both day-gaps; if reviewer-accepted, note that", "evidence": [] },
    { "id": "CR-5", "label": "Prior-claims pattern",       "result": "pass | fail | n/a", "severity": "info | warn | critical", "detail": "string", "evidence": [] }
  ],
  "findings": [ "string" ],
  "summary":  "string — 2 to 6 sentences",
  "details": {
    "overall_risk":   "low | medium | high",
    "dimension_risk": {
      "narrative":    "low | medium | high",
      "estimate":     "low | medium | high",
      "documentation":"low | medium | high",
      "timing":       "low | medium | high",
      "prior_claims": "low | medium | high"
    },
    "risk_indicators":  [ { "dimension": "narrative | estimate | documentation | timing | prior_claims", "description": "string" } ],
    "reasonableness": {
      "claim_total":       0.00,
      "assessor_estimate": 0.00,
      "ratio":             0.00,
      "currency":          "string"
    },
    "submission_to_incident_days":  0,
    "assessment_to_incident_days":  0,
    "prior_claims_summary":         { "count": 0, "same_type_count_24mo": 0, "denied_count": 0 }
  }
}
```

### Output JSON

```json
{
  "type": "object",
  "required": [ "out_CredibilityAssessmentJSON" ],
  "properties": {
    "out_CredibilityAssessmentJSON": {
      "type": "object",
      "description": "Unified envelope with checks CR-1..CR-5 plus a details block containing per-dimension risk levels, an aggregated overall_risk, the reasonableness ratio (mirroring P-8), date-gap counts, and prior-claims summary. status is derived strictly from check severities. Recommendation is escalate_to_human unless overall_risk is low.",
      "properties": {}
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: India water-damage claim. Ratio 1.314 (medium estimate risk). Timing was 95 days but reviewer accepted hospitalisation explanation → CR-4 downgraded to low. No prior claims.

#### Inputs

```json
{
  "in_ClaimDataJSON": {
    "Claim":          [ { "ClaimID": "CLM-2026-357861", "DateOfSubmission": "26/05/2026" } ],
    "ClaimIncident":  [ { "DateOfIncident": "20/02/2026", "TypeOfIncident": "Water Damage", "DescriptionOfIncident": "Burst pipe in kitchen flooded ground floor. Claimant was hospitalised between 21/02 and 23/02 with pneumonia; submission delayed accordingly." } ],
    "ClaimClaimTotals": [ { "TotalClaimAmount": "₹335,000.00" } ]
  },
  "in_PolicyDataJSON":           { "coverage_summary": { "currency": "INR" } },
  "in_AssessmentReportJSON": {
    "report_metadata": { "assessment_date": "23/05/2026" },
    "findings": { "narrative": "On-site inspection confirmed extensive water damage to ground-floor kitchen", "cause_determination": "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident.", "damage_scope_description": "Engineered timber flooring throughout 25 sqm of kitchen and 12 sqm of dining room is warped and unsalvageable." },
    "cost_summary": { "total_estimated_repair_cost": "₹255,000.00" }
  },
  "in_ClaimEligibilityJSON":     { "claim_id": "CLM-2026-357861", "recommendation": "escalate" },
  "in_AssessmentValidationJSON": { "claim_id": "CLM-2026-357861", "recommendation": "proceed_to_parallel_analysis" },
  "in_PriorClaimsJSON":          [],
  "in_EscalationDecision_Eligibility": "continue",
  "in_EscalationComment_Eligibility":  "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis."
}
```

#### Outputs

```json
{
  "out_CredibilityAssessmentJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "escalate_to_human",
    "checks": [
      { "id": "CR-1", "label": "Narrative consistency", "result": "pass", "severity": "info",
        "detail": "Narratives are consistent (per-dimension risk: low). Claimant reports a burst pipe in the kitchen; assessor confirms failure of a copper feeder pipe behind the kitchen sink.",
        "evidence": [
          { "source": "claim.ClaimIncident[0].DescriptionOfIncident", "value": "Burst pipe in kitchen flooded ground floor ..." },
          { "source": "assessment.findings.cause_determination",      "value": "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident." }
        ] },

      { "id": "CR-2", "label": "Estimate reasonableness", "result": "pass", "severity": "warn",
        "detail": "Ratio = ₹335,000.00 / ₹255,000.00 = 1.314 (per-dimension risk: medium). Claimant claiming 31.4 % above the independent assessor estimate.",
        "evidence": [
          { "source": "claim.ClaimClaimTotals[0].TotalClaimAmount",         "value": "₹335,000.00" },
          { "source": "assessment.cost_summary.total_estimated_repair_cost","value": "₹255,000.00" }
        ] },

      { "id": "CR-3", "label": "Documentation completeness", "result": "pass", "severity": "info",
        "detail": "All required FNOL fields populated. Assessor's report carries license, signature, and dated assessment. Damage inventory reconciles to claim total within tolerance. (per-dimension risk: low)",
        "evidence": [] },

      { "id": "CR-4", "label": "Timing and pattern", "result": "pass", "severity": "info",
        "detail": "Submission-to-incident gap is 95 days (would normally be flagged), but the human reviewer at the eligibility stage already accepted this explanation citing documented hospitalisation. Assessment-to-incident gap is 92 days — assessment well after the incident but not so long as to invalidate observations. Reviewer-accepted; per-dimension risk: low.",
        "evidence": [
          { "source": "claim.ClaimIncident[0].DateOfIncident",        "value": "20/02/2026" },
          { "source": "claim.Claim[0].DateOfSubmission",              "value": "26/05/2026" },
          { "source": "assessment.report_metadata.assessment_date",   "value": "23/05/2026" },
          { "source": "review.eligibility.comment",                   "value": "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis." }
        ] },

      { "id": "CR-5", "label": "Prior-claims pattern", "result": "pass", "severity": "info",
        "detail": "No prior-claims data available. Per-dimension risk: low.",
        "evidence": [ { "source": "in_PriorClaimsJSON", "value": "[] (empty)" } ] }
    ],
    "findings": [
      "Overall risk: medium — driven by an elevated claim-to-estimate ratio (1.314).",
      "Narratives are consistent between claimant and assessor.",
      "Documentation is complete; no anomalies in the FNOL or the assessor's report.",
      "Timing concerns at the eligibility stage were already accepted by the human reviewer."
    ],
    "summary":  "The claim shows one elevated credibility dimension: the estimate reasonableness ratio is 1.314, putting the claimant's total 31.4 % above the assessor's independent estimate. All other dimensions (narrative consistency, documentation completeness, timing, prior-claims pattern) are low risk. Overall risk is therefore medium. Per the aggregation rule, this routes to escalate_to_human so a senior adjuster can decide whether to settle at the claimant's amount, the assessor's amount, or somewhere in between. The timing concern at the eligibility stage has already been accepted by the reviewer on documented hospitalisation grounds and is not re-raised here.",
    "details": {
      "overall_risk":   "medium",
      "dimension_risk": {
        "narrative":    "low",
        "estimate":     "medium",
        "documentation":"low",
        "timing":       "low",
        "prior_claims": "low"
      },
      "risk_indicators": [
        { "dimension": "estimate", "description": "Ratio of claim total to assessor estimate is 1.314, in the 1.20–1.50 medium-risk band." }
      ],
      "reasonableness": {
        "claim_total":      335000.00,
        "assessor_estimate": 255000.00,
        "ratio":            1.314,
        "currency":         "INR"
      },
      "submission_to_incident_days":  95,
      "assessment_to_incident_days":  92,
      "prior_claims_summary":         { "count": 0, "same_type_count_24mo": 0, "denied_count": 0 }
    }
  }
}
```
