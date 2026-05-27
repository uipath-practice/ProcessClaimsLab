### Inputs Prompt
- in_ClaimDataJSON: Cleaned claim data emitted by the 01 Claim Eligibility Agent. Same shape as that agent's output.
- in_PolicyDataJSON: Structured policy data emitted by the 01 Claim Eligibility Agent.
- in_AssessorReportPDF: The assessor's incident report PDF, passed as an Orchestrator job attachment.
- in_ClaimEligibilityJSON: The eligibility envelope from Agent 01, for context (recommendation, summary, findings, checks E-1..E-5).
- in_EscalationDecision_Eligibility: Optional. Present when a human reviewer has overridden at the eligibility stage. Value is `"continue"` or `"deny"`. Honour the human decision.
- in_EscalationComment_Eligibility: Optional. The reviewer's free-text notes explaining the override.

### Inputs JSON

```json
{
  "type": "object",
  "required": [
    "in_ClaimDataJSON",
    "in_PolicyDataJSON",
    "in_AssessorReportPDF",
    "in_ClaimEligibilityJSON"
  ],
  "properties": {
    "in_ClaimDataJSON":         { "type": "object", "description": "Cleaned claim data from Agent 01.", "properties": {} },
    "in_PolicyDataJSON":        { "type": "object", "description": "Structured policy data from Agent 01.", "properties": {} },
    "in_AssessorReportPDF": {
      "description": "PDF of the assessor's incident report, passed as an Orchestrator job attachment.",
      "$ref": "#/definitions/job-attachment"
    },
    "in_ClaimEligibilityJSON":  { "type": "object", "description": "Eligibility envelope from Agent 01 — context for downstream reasoning.", "properties": {} },
    "in_EscalationDecision_Eligibility": {
      "type": "string",
      "description": "Optional. 'continue' or 'deny' when a human reviewer has overridden the eligibility result. Honour the human decision in your output."
    },
    "in_EscalationComment_Eligibility": {
      "type": "string",
      "description": "Optional. The reviewer's free-text notes explaining the eligibility override. Do not re-raise concerns the reviewer has addressed."
    }
  },
  "definitions": {
    "job-attachment": {
      "type": "object",
      "required": ["ID"],
      "x-uipath-resource-kind": "JobAttachment",
      "properties": {
        "ID":       { "type": "string", "description": "Orchestrator attachment key" },
        "FullName": { "type": "string", "description": "File name" },
        "MimeType": { "type": "string", "description": "Content MIME type, e.g. application/pdf" },
        "Metadata": {
          "type": "object",
          "description": "Free-form string-keyed metadata",
          "additionalProperties": { "type": "string" }
        }
      }
    }
  },
  "title": "Inputs"
}
```

### Output Format

The Assessment Validation Agent produces two outputs.

**`out_AssessmentValidationJSON`** — unified envelope. Structure:

```json
{
  "claim_id":       "string",
  "status":         "ok | warn | critical",
  "recommendation": "proceed_to_parallel_analysis | escalate | reject_report",
  "checks": [
    {
      "id":       "AV-1",
      "label":    "Document identity",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "assessment.report_metadata.report_reference", "value": "string" },
        { "source": "claim.Claim[0].ClaimID",                      "value": "string" }
      ]
    },
    {
      "id":       "AV-2",
      "label":    "Required fields present",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string — list missing fields if any",
      "evidence": [
        { "source": "assessment.report_metadata.assessor.license",   "value": "string or null" },
        { "source": "assessment.findings.cause_determination",       "value": "string or null" },
        { "source": "assessment.cost_summary.total_estimated_repair_cost", "value": "string or null" }
      ]
    },
    {
      "id":       "AV-3",
      "label":    "Assessor credentials",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "assessment.report_metadata.assessor.name",     "value": "string" },
        { "source": "assessment.report_metadata.assessor.license",  "value": "string or null" }
      ]
    },
    {
      "id":       "AV-4",
      "label":    "Property and claim match",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string — describe each mismatch if any",
      "evidence": [
        { "source": "assessment.property.address",         "value": "string" },
        { "source": "claim.ClaimProperty[0]",              "value": "string" },
        { "source": "assessment.property.claim_reference", "value": "string" },
        { "source": "claim.Claim[0].ClaimID",              "value": "string" }
      ]
    },
    {
      "id":       "AV-5",
      "label":    "Internal consistency",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string — describe each inconsistency if any",
      "evidence": [
        { "source": "assessment.repair_schedule",                              "value": "<line items summary>" },
        { "source": "assessment.cost_summary.total_estimated_repair_cost",    "value": "string" }
      ]
    },
    {
      "id":       "AV-6",
      "label":    "Assessment / claim peril alignment",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "claim.ClaimIncident[0].TypeOfIncident",   "value": "string" },
        { "source": "assessment.findings.cause_determination", "value": "string" }
      ]
    }
  ],
  "findings": [ "string — 1 to 5 short bullets" ],
  "summary":  "string — 2 to 6 sentence plain-text paragraph",
  "details": {
    "report_usability":           "usable | usable_with_flags | not_usable",
    "report_complete":            true,
    "assessor_credentials_valid": true,
    "peril_aligned":              true,
    "missing_fields":             [],
    "mismatches":                 [],
    "internal_inconsistencies":   []
  }
}
```

**`out_AssessmentReportJSON`** — structured report data. Match this structure exactly:

```json
{
  "report_metadata": {
    "report_reference": "string",
    "assessor": {
      "name":    "string",
      "license": "string or null",
      "company": "string"
    },
    "assessment_date": "string — DD/MM/YYYY"
  },
  "property": {
    "address": { "line1": "string", "city": "string", "state": "string", "zip": "string" },
    "policyholder_name":      "string",
    "claim_reference":        "string",
    "incident_date":          "string — DD/MM/YYYY",
    "reported_incident_type": "string"
  },
  "findings": {
    "narrative":                "string",
    "cause_determination":      "string",
    "damage_scope_description": "string"
  },
  "repair_schedule": [
    { "description": "string", "estimated_cost": "string — formatted currency with symbol" }
  ],
  "cost_summary": {
    "total_estimated_repair_cost": "string — formatted currency with symbol"
  },
  "assessor_opinion": "string"
}
```

### Output JSON

```json
{
  "type": "object",
  "required": [
    "out_AssessmentValidationJSON",
    "out_AssessmentReportJSON"
  ],
  "properties": {
    "out_AssessmentValidationJSON": {
      "type": "object",
      "description": "Unified envelope with checks AV-1..AV-6, plus a details block describing usability, missing fields, and inconsistencies. status is derived strictly from check severities.",
      "properties": {}
    },
    "out_AssessmentReportJSON": {
      "type": "object",
      "description": "Structured representation of the assessor's incident report. Missing fields are null, not guessed. Monetary amounts kept as formatted strings with original currency symbols.",
      "properties": {}
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: synthetic-document generator, India locale (INR). Assessor report aligns well with claim — clean validation outcome.

#### Inputs

```json
{
  "in_ClaimDataJSON": {
    "Claim":         [ { "ClaimID": "CLM-2026-357861", "InsurerName": "Apex Mutual Insurance Company", "InsurerAddress": "Claims Department, P.O. Box 9400, Hartford, CT 06152", "DateOfSubmission": "26/05/2026" } ],
    "ClaimClaimant": [ { "Name": "Rajesh Kumar Sharma", "PhoneNumber": "+91 98765 43210", "EmailAddress": "rajesh.sharma@example.in", "PolicyNumber": "HO-5534416561" } ],
    "ClaimProperty": [ { "StreetAddress": "12 MG Road, Apt 4B", "City": "Bengaluru", "State": "Karnataka", "ZipCode": "560001", "IsPrimary": true, "IsPresent": true } ],
    "ClaimIncident": [ { "DateOfIncident": "20/02/2026", "TimeOfIncident": "approximately 22:30", "TypeOfIncident": "Water Damage", "DescriptionOfIncident": "Burst pipe in kitchen flooded ground floor.", "PoliceReportFiled": false, "PoliceReportNumber": null, "EmergencyServicesCalled": false, "TemporaryRepairsMade": true, "TemporaryRepairsDescription": "Main water shutoff closed; affected area dried with fans for 24h." } ]
  },
  "in_PolicyDataJSON": {
    "declarations": { "policy_number": "HO-5534416561", "named_insured": "Rajesh Kumar Sharma", "described_property": { "address": { "line1": "12 MG Road, Apt 4B", "city": "Bengaluru", "state": "Karnataka", "pin": "560001" } } },
    "coverage_summary": { "currency": "INR" }
  },
  "in_AssessorReportPDF": {
    "MimeType": "application/pdf",
    "FullName": "CLM-2026-357861-incident.pdf",
    "ID":       "5f9040d3-e938-4671-4add-08de9ae210e3",
    "Metadata": {}
  },
  "in_ClaimEligibilityJSON": {
    "claim_id": "CLM-2026-357861",
    "status":   "warn",
    "recommendation": "escalate",
    "findings": [ "Filing was 95 days after the incident — beyond the 60-day deadline." ],
    "summary":  "Eligible subject to human review of the late filing."
  },
  "in_EscalationDecision_Eligibility": "continue",
  "in_EscalationComment_Eligibility":  "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis."
}
```

#### Outputs

```json
{
  "out_AssessmentValidationJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "ok",
    "recommendation": "proceed_to_parallel_analysis",
    "checks": [
      { "id": "AV-1", "label": "Document identity", "result": "pass", "severity": "info",
        "detail": "Report is an assessor incident report and the claim reference matches.",
        "evidence": [
          { "source": "assessment.report_metadata.report_reference", "value": "RPT-CLM-2026-357861" },
          { "source": "claim.Claim[0].ClaimID",                      "value": "CLM-2026-357861" }
        ] },

      { "id": "AV-2", "label": "Required fields present", "result": "pass", "severity": "info",
        "detail": "All required fields present in the report.",
        "evidence": [
          { "source": "assessment.report_metadata.assessor.license",        "value": "IRDAI-LP-12345" },
          { "source": "assessment.findings.cause_determination",            "value": "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet." },
          { "source": "assessment.cost_summary.total_estimated_repair_cost","value": "₹255,000.00" }
        ] },

      { "id": "AV-3", "label": "Assessor credentials", "result": "pass", "severity": "info",
        "detail": "License IRDAI-LP-12345; assessor identity consistent across header and certification.",
        "evidence": [
          { "source": "assessment.report_metadata.assessor.name",    "value": "Vikram Patel" },
          { "source": "assessment.report_metadata.assessor.license", "value": "IRDAI-LP-12345" }
        ] },

      { "id": "AV-4", "label": "Property and claim match", "result": "pass", "severity": "info",
        "detail": "Address, policyholder name, claim reference, and incident date all match.",
        "evidence": [
          { "source": "assessment.property.address",         "value": "12 MG Road, Apt 4B, Bengaluru, Karnataka 560001" },
          { "source": "claim.ClaimProperty[0]",              "value": "12 MG Road, Apt 4B, Bengaluru, Karnataka 560001" },
          { "source": "assessment.property.claim_reference", "value": "CLM-2026-357861" },
          { "source": "claim.Claim[0].ClaimID",              "value": "CLM-2026-357861" }
        ] },

      { "id": "AV-5", "label": "Internal consistency", "result": "pass", "severity": "info",
        "detail": "Sum of repair-schedule line items (₹255,000) matches the stated total.",
        "evidence": [
          { "source": "assessment.repair_schedule",                          "value": "5 line items summing to ₹255,000.00" },
          { "source": "assessment.cost_summary.total_estimated_repair_cost", "value": "₹255,000.00" }
        ] },

      { "id": "AV-6", "label": "Assessment / claim peril alignment", "result": "pass", "severity": "info",
        "detail": "Claimant reports 'Water Damage'; assessor confirms burst-pipe cause. Aligned.",
        "evidence": [
          { "source": "claim.ClaimIncident[0].TypeOfIncident",   "value": "Water Damage" },
          { "source": "assessment.findings.cause_determination", "value": "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident." }
        ] }
    ],
    "findings": [
      "All six validation checks passed.",
      "Assessor identity and credentials are present.",
      "Repair-schedule total reconciles to the stated total."
    ],
    "summary":  "The assessor's report is complete, internally consistent, and aligned with the claim. Property, policyholder, claim reference, and incident date all match. The cause determination (burst copper feeder pipe) supports the claimant's reported water damage. The report is ready for parallel coverage, payout, and credibility analysis. Note: a human reviewer accepted the late filing at the eligibility stage citing the claimant's hospitalisation; no validation issue arises from that prior decision.",
    "details": {
      "report_usability":           "usable",
      "report_complete":            true,
      "assessor_credentials_valid": true,
      "peril_aligned":              true,
      "missing_fields":             [],
      "mismatches":                 [],
      "internal_inconsistencies":   []
    }
  },
  "out_AssessmentReportJSON": {
    "report_metadata": {
      "report_reference": "RPT-CLM-2026-357861",
      "assessor": { "name": "Vikram Patel", "license": "IRDAI-LP-12345", "company": "Patel & Associates Surveyors Pvt. Ltd." },
      "assessment_date": "23/05/2026"
    },
    "property": {
      "address": { "line1": "12 MG Road, Apt 4B", "city": "Bengaluru", "state": "Karnataka", "zip": "560001" },
      "policyholder_name":      "Rajesh Kumar Sharma",
      "claim_reference":        "CLM-2026-357861",
      "incident_date":          "20/02/2026",
      "reported_incident_type": "Water Damage"
    },
    "findings": {
      "narrative":                "On-site inspection conducted on 23/05/2026 confirmed extensive water damage to ground-floor kitchen, dining room, and adjacent passage flooring.",
      "cause_determination":      "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident.",
      "damage_scope_description": "Engineered timber flooring throughout 25 sqm of kitchen and 12 sqm of dining room is warped and unsalvageable. Drywall to a height of 30 cm above floor level is saturated. Kitchen base cabinets are water-damaged."
    },
    "repair_schedule": [
      { "description": "Remove and replace engineered timber flooring (37 sqm)",   "estimated_cost": "₹110,000.00" },
      { "description": "Drywall replacement to 30 cm above floor level (perimeter)","estimated_cost": "₹25,000.00" },
      { "description": "Kitchen base cabinet replacement (3 units)",                "estimated_cost": "₹75,000.00" },
      { "description": "Plumbing repair — replace failed copper feeder section",    "estimated_cost": "₹15,000.00" },
      { "description": "Industrial drying and dehumidification (5 days)",           "estimated_cost": "₹30,000.00" }
    ],
    "cost_summary": { "total_estimated_repair_cost": "₹255,000.00" },
    "assessor_opinion": "The damage observed is consistent in scope and pattern with the reported burst pipe event on 20/02/2026. The claimant took reasonable mitigating steps. The estimated repair cost is in line with current Bengaluru market rates."
  }
}
```
