### Inputs Prompt
- in_ClaimIXPDataJSON: The First Notice of Loss claim form, as a raw JSON object produced by IXP extraction. Each field is wrapped with `Value`, `Confidence`, and `OcrConfidence` metadata.
- in_PolicyID: The HO-3 homeowner policy number referenced on the claim form.
- in_PolicyPDF: The full insurance policy PDF passed as a UiPath Orchestrator job attachment.

### Inputs JSON

```json
{
  "type": "object",
  "required": [
    "in_PolicyID",
    "in_ClaimIXPDataJSON",
    "in_PolicyPDF"
  ],
  "properties": {
    "in_PolicyID": {
      "type": "string",
      "description": "ID of the insurance policy referenced on the FNOL form"
    },
    "in_ClaimIXPDataJSON": {
      "type": "object",
      "description": "Raw IXP extraction of the claim form, including Confidence and OcrConfidence on every field.",
      "properties": {}
    },
    "in_PolicyPDF": {
      "description": "PDF of the insurance policy associated with the claim, passed as an Orchestrator job attachment.",
      "$ref": "#/definitions/job-attachment"
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

The Claim Eligibility Agent produces four outputs.

**`out_ClaimEligibilityJSON`** — the unified envelope. Structure:

```json
{
  "claim_id": "string — the ClaimID from the claim form",
  "status":   "ok | warn | critical",
  "recommendation": "proceed_to_coverage_analysis | deny | escalate",
  "checks": [
    {
      "id":       "E-1",
      "label":    "Policy in force",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string — one-sentence outcome",
      "evidence": [
        { "source": "policy.declarations.payment_status", "value": "string — observed value" }
      ]
    },
    {
      "id":       "E-2",
      "label":    "Identity match",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "claim.ClaimClaimant[0].Name",        "value": "string" },
        { "source": "policy.declarations.named_insured",  "value": "string" }
      ]
    },
    {
      "id":       "E-3",
      "label":    "Property address match",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "claim.ClaimProperty[0]",                          "value": "string — un-normalised claim address" },
        { "source": "policy.declarations.described_property.address",  "value": "string — un-normalised policy address" }
      ]
    },
    {
      "id":       "E-4",
      "label":    "Coverage period",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string",
      "evidence": [
        { "source": "claim.ClaimIncident[0].DateOfIncident",      "value": "string — DD/MM/YYYY" },
        { "source": "claim.Claim[0].DateOfSubmission",            "value": "string — DD/MM/YYYY" },
        { "source": "policy.declarations.policy_period.effective_date",  "value": "string — DD/MM/YYYY" },
        { "source": "policy.declarations.policy_period.expiration_date", "value": "string — DD/MM/YYYY" }
      ]
    },
    {
      "id":       "E-5",
      "label":    "Filing deadline",
      "result":   "pass | fail | n/a",
      "severity": "info | warn | critical",
      "detail":   "string — include 'late_with_justification' if applicable, plus the days_elapsed value",
      "evidence": [
        { "source": "claim.ClaimIncident[0].DateOfIncident",     "value": "string — DD/MM/YYYY" },
        { "source": "claim.Claim[0].DateOfSubmission",           "value": "string — DD/MM/YYYY" }
      ]
    }
  ],
  "findings": [ "string — 1 to 5 short bullets suitable for human display" ],
  "summary":  "string — 2 to 6 sentence plain-text paragraph",
  "details": {
    "is_eligible_strict": true,
    "missing_documents":  [],
    "policy_currency":    "INR"
  }
}
```

**`out_ClaimDataJSON`** — cleaned claim data, no confidence metadata. Match this structure exactly:

```json
{
  "Claim": [
    {
      "ClaimID":          "string",
      "InsurerName":      "string — extracted from the claim form header / branding block",
      "InsurerAddress":   "string — extracted from the claim form header / branding block",
      "DateOfSubmission": "string — DD/MM/YYYY"
    }
  ],
  "ClaimClaimant": [
    {
      "Name":         "string",
      "PhoneNumber":  "string",
      "EmailAddress": "string",
      "PolicyNumber": "string"
    }
  ],
  "ClaimProperty": [
    {
      "StreetAddress": "string",
      "City":          "string",
      "State":         "string",
      "ZipCode":       "string",
      "IsPrimary":     true,
      "IsPresent":     true
    }
  ],
  "ClaimIncident": [
    {
      "DateOfIncident":              "string — DD/MM/YYYY",
      "TimeOfIncident":              "string",
      "TypeOfIncident":              "string",
      "DescriptionOfIncident":       "string",
      "PoliceReportFiled":           true,
      "PoliceReportNumber":          "string or null",
      "EmergencyServicesCalled":     true,
      "TemporaryRepairsMade":        true,
      "TemporaryRepairsDescription": "string"
    }
  ],
  "ClaimDamageInventory": [
    {
      "Category":        "string",
      "Location":        "string",
      "Description":     "string",
      "EstimatedCost":   "string — formatted currency with symbol, e.g. '₹125,000.00'",
      "RepairOrReplace": "string"
    }
  ],
  "ClaimClaimTotals": [
    {
      "TotalStructureDamage":          "string — formatted currency with symbol",
      "TotalPersonalProperty":         "string — formatted currency with symbol",
      "TotalAdditionalLivingExpenses": "string — formatted currency with symbol",
      "TotalClaimAmount":              "string — formatted currency with symbol"
    }
  ]
}
```

**`out_PolicyDataJSON`** — full structured policy. Match this structure exactly:

```json
{
  "insurer": {
    "name":    "string",
    "address": "string"
  },
  "policy_form": {
    "form_type": "string",
    "form_code": "string"
  },
  "declarations": {
    "policy_number":    "string",
    "policy_type":      "string",
    "named_insured":    "string",
    "mailing_address":  { "line1": "string", "city": "string", "state": "string", "pin": "string" },
    "policy_period":    { "effective_date": "string — DD/MM/YYYY", "expiration_date": "string — DD/MM/YYYY" },
    "described_property": {
      "address":        { "line1": "string", "city": "string", "state": "string", "pin": "string" },
      "property_type":  "string",
      "year_built":     0,
      "square_footage": "string"
    },
    "annual_premium":   "string — formatted currency with symbol",
    "payment_status":   "string",
    "producing_agent":  { "name": "string", "phone": "string" }
  },
  "coverage_summary": {
    "currency": "string — ISO 4217 code (e.g. INR, USD, HKD, EUR, GBP)",
    "coverages": [
      { "coverage_code": "A | B | C | D | E | F", "description": "string", "limit_of_liability": "string — formatted currency", "deductible": "string — formatted currency or 'None'" }
    ],
    "special_limits_of_liability_coverage_c": [
      { "property_category": "string", "sublimit": "string — formatted currency" }
    ]
  },
  "insuring_agreement": { "text": "string" },
  "section_I_covered_perils": {
    "coverage_A_and_B": { "type": "string", "description": "string" },
    "coverage_C": {
      "type":   "Named Perils",
      "perils": [ { "number": 0, "name": "string" } ]
    }
  },
  "section_II_exclusions": {
    "A_excluded_perils": [ { "number": 0, "name": "string" } ],
    "B_other_exclusions": [ { "number": 0, "name": "string" } ]
  },
  "section_III_special_conditions": [ { "number": 0, "name": "string", "value": "string (optional)" } ],
  "section_IV_endorsements": [ { "endorsement_number": 0, "name": "string", "status": "Included | Not Included" } ],
  "section_V_general_conditions": [ { "number": 0, "name": "string" } ]
}
```

**`out_isEligible`** — boolean. `true` if every check passes (including `late_with_justification` on E-5), `false` if any critical fail.

### Output JSON

```json
{
  "type": "object",
  "required": [
    "out_ClaimEligibilityJSON",
    "out_ClaimDataJSON",
    "out_PolicyDataJSON",
    "out_isEligible"
  ],
  "properties": {
    "out_ClaimEligibilityJSON": {
      "type": "object",
      "description": "Unified envelope output with claim_id, status, recommendation, checks E-1..E-5, findings, summary, details. status is derived strictly from check severities.",
      "properties": {}
    },
    "out_ClaimDataJSON": {
      "type": "object",
      "description": "Cleaned claim data. Confidence and OcrConfidence metadata stripped. Monetary fields kept as formatted strings with original currency symbols.",
      "properties": {}
    },
    "out_PolicyDataJSON": {
      "type": "object",
      "description": "Full structured representation of the HO-3 policy PDF. Every coverage, sublimit, exclusion, special condition, and endorsement preserved. coverage_summary.currency is ISO 4217.",
      "properties": {}
    },
    "out_isEligible": {
      "type": "boolean",
      "description": "True if every check passes; false if any check fails with severity critical."
    }
  },
  "title": "Outputs"
}
```

### Sample Payload

**Source**: synthetic-document generator profile, India locale (INR), late-filing discrepancy applied.

#### Inputs

```json
{
  "in_PolicyID": "HO-5534416561",
  "in_ClaimIXPDataJSON": {
    "Claim": [
      {
        "ClaimID":          { "Value": "CLM-2026-357861",           "Confidence": 0.9999, "OcrConfidence": 1.0 },
        "InsurerName":      { "Value": "Apex Mutual Insurance Company", "Confidence": 0.998, "OcrConfidence": 1.0 },
        "InsurerAddress":   { "Value": "Claims Department, P.O. Box 9400, Hartford, CT 06152", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "DateOfSubmission": { "Value": "26/05/2026",                "Confidence": 0.97, "OcrConfidence": 1.0 }
      }
    ],
    "ClaimClaimant": [
      {
        "Name":         { "Value": "Rajesh Kumar Sharma",       "Confidence": 0.99, "OcrConfidence": 1.0 },
        "PhoneNumber":  { "Value": "+91 98765 43210",           "Confidence": 0.99, "OcrConfidence": 1.0 },
        "EmailAddress": { "Value": "rajesh.sharma@example.in",  "Confidence": 0.99, "OcrConfidence": 1.0 },
        "PolicyNumber": { "Value": "HO-5534416561",             "Confidence": 0.99, "OcrConfidence": 1.0 }
      }
    ],
    "ClaimProperty": [
      {
        "StreetAddress": { "Value": "12 MG Road, Apt 4B", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "City":          { "Value": "Bengaluru",           "Confidence": 0.99, "OcrConfidence": 1.0 },
        "State":         { "Value": "Karnataka",           "Confidence": 0.99, "OcrConfidence": 1.0 },
        "ZipCode":       { "Value": "560001",              "Confidence": 0.99, "OcrConfidence": 1.0 },
        "IsPrimary":     { "Value": "True",                "Confidence": 0.99, "OcrConfidence": -1.0 },
        "IsPresent":     { "Value": "True",                "Confidence": 0.99, "OcrConfidence": -1.0 }
      }
    ],
    "ClaimIncident": [
      {
        "DateOfIncident":              { "Value": "20/02/2026", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TimeOfIncident":              { "Value": "approximately 22:30", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TypeOfIncident":              { "Value": "Water Damage", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "DescriptionOfIncident":       { "Value": "Burst pipe in kitchen flooded ground floor. Claimant was hospitalised between 21/02 and 23/02 with pneumonia; submission delayed accordingly.", "Confidence": 0.92, "OcrConfidence": 1.0 },
        "PoliceReportFiled":           { "Value": "False", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "PoliceReportNumber":          { "Value": "N/A",   "Confidence": 0.99, "OcrConfidence": 1.0 },
        "EmergencyServicesCalled":     { "Value": "False", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TemporaryRepairsMade":        { "Value": "True",  "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TemporaryRepairsDescription": { "Value": "Main water shutoff closed; affected area dried with fans for 24h.", "Confidence": 0.98, "OcrConfidence": 1.0 }
      }
    ],
    "ClaimDamageInventory": [
      { "Category": { "Value": "Structure - Flooring", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "Location": { "Value": "Kitchen",              "Confidence": 0.99, "OcrConfidence": 1.0 },
        "Description":   { "Value": "Engineered timber flooring warped from water exposure, 25 sqm affected", "Confidence": 0.95, "OcrConfidence": 1.0 },
        "EstimatedCost": { "Value": "₹125,000.00", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "RepairOrReplace": { "Value": "Replace", "Confidence": 0.99, "OcrConfidence": 1.0 } }
    ],
    "ClaimClaimTotals": [
      {
        "TotalStructureDamage":          { "Value": "₹275,000.00", "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TotalPersonalProperty":         { "Value": "₹60,000.00",  "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TotalAdditionalLivingExpenses": { "Value": "₹0.00",       "Confidence": 0.99, "OcrConfidence": 1.0 },
        "TotalClaimAmount":              { "Value": "₹335,000.00", "Confidence": 0.99, "OcrConfidence": 1.0 }
      }
    ]
  },
  "in_PolicyPDF": {
    "MimeType":  "application/pdf",
    "FullName":  "HO-5534416561.pdf",
    "ID":        "5f9040d3-e938-4671-4add-08de9ae210e2",
    "Metadata":  {}
  }
}
```

#### Outputs

```json
{
  "out_isEligible": true,
  "out_ClaimEligibilityJSON": {
    "claim_id":       "CLM-2026-357861",
    "status":         "warn",
    "recommendation": "escalate",
    "checks": [
      {
        "id": "E-1", "label": "Policy in force", "result": "pass", "severity": "info",
        "detail": "Payment status: Current - Paid in Full",
        "evidence": [{ "source": "policy.declarations.payment_status", "value": "Current - Paid in Full" }]
      },
      {
        "id": "E-2", "label": "Identity match", "result": "pass", "severity": "info",
        "detail": "Claimant name matches named insured exactly.",
        "evidence": [
          { "source": "claim.ClaimClaimant[0].Name",        "value": "Rajesh Kumar Sharma" },
          { "source": "policy.declarations.named_insured",  "value": "Rajesh Kumar Sharma" }
        ]
      },
      {
        "id": "E-3", "label": "Property address match", "result": "pass", "severity": "info",
        "detail": "Claim and policy property addresses match after normalisation.",
        "evidence": [
          { "source": "claim.ClaimProperty[0]",                          "value": "12 MG Road, Apt 4B, Bengaluru, Karnataka 560001" },
          { "source": "policy.declarations.described_property.address",  "value": "12 MG Road, Apt 4B, Bengaluru, Karnataka 560001" }
        ]
      },
      {
        "id": "E-4", "label": "Coverage period", "result": "pass", "severity": "info",
        "detail": "Incident date (20/02/2026) falls within policy period (01/04/2025 to 31/03/2026).",
        "evidence": [
          { "source": "claim.ClaimIncident[0].DateOfIncident",                "value": "20/02/2026" },
          { "source": "claim.Claim[0].DateOfSubmission",                      "value": "26/05/2026" },
          { "source": "policy.declarations.policy_period.effective_date",     "value": "01/04/2025" },
          { "source": "policy.declarations.policy_period.expiration_date",    "value": "31/03/2026" }
        ]
      },
      {
        "id": "E-5", "label": "Filing deadline", "result": "pass", "severity": "warn",
        "detail": "late_with_justification — 95 days elapsed; claimant cites hospitalisation between 21/02 and 23/02 as cause of delay.",
        "evidence": [
          { "source": "claim.ClaimIncident[0].DateOfIncident", "value": "20/02/2026" },
          { "source": "claim.Claim[0].DateOfSubmission",       "value": "26/05/2026" }
        ]
      }
    ],
    "findings": [
      "All four hard eligibility checks (E-1..E-4) passed.",
      "Filing was 95 days after the incident — beyond the 60-day deadline.",
      "Claimant provides a hospitalisation explanation in the incident narrative."
    ],
    "summary":  "The claim is eligible to proceed, subject to human review of the late filing. Policy is in force; identity and property address match; the loss occurred within the policy period. The submission was 95 days after the incident, exceeding the 60-day filing deadline, but the claimant cites hospitalisation as the reason for the delay. Per the filing deadline rule, this routes to an escalation for human determination of whether the justification is acceptable.",
    "details": {
      "is_eligible_strict": true,
      "missing_documents":  [],
      "policy_currency":    "INR"
    }
  },
  "out_ClaimDataJSON": {
    "Claim": [
      { "ClaimID": "CLM-2026-357861", "InsurerName": "Apex Mutual Insurance Company", "InsurerAddress": "Claims Department, P.O. Box 9400, Hartford, CT 06152", "DateOfSubmission": "26/05/2026" }
    ],
    "ClaimClaimant": [
      { "Name": "Rajesh Kumar Sharma", "PhoneNumber": "+91 98765 43210", "EmailAddress": "rajesh.sharma@example.in", "PolicyNumber": "HO-5534416561" }
    ],
    "ClaimProperty": [
      { "StreetAddress": "12 MG Road, Apt 4B", "City": "Bengaluru", "State": "Karnataka", "ZipCode": "560001", "IsPrimary": true, "IsPresent": true }
    ],
    "ClaimIncident": [
      {
        "DateOfIncident": "20/02/2026", "TimeOfIncident": "approximately 22:30", "TypeOfIncident": "Water Damage",
        "DescriptionOfIncident": "Burst pipe in kitchen flooded ground floor. Claimant was hospitalised between 21/02 and 23/02 with pneumonia; submission delayed accordingly.",
        "PoliceReportFiled": false, "PoliceReportNumber": null, "EmergencyServicesCalled": false,
        "TemporaryRepairsMade": true, "TemporaryRepairsDescription": "Main water shutoff closed; affected area dried with fans for 24h."
      }
    ],
    "ClaimDamageInventory": [
      { "Category": "Structure - Flooring", "Location": "Kitchen", "Description": "Engineered timber flooring warped from water exposure, 25 sqm affected", "EstimatedCost": "₹125,000.00", "RepairOrReplace": "Replace" }
    ],
    "ClaimClaimTotals": [
      { "TotalStructureDamage": "₹275,000.00", "TotalPersonalProperty": "₹60,000.00", "TotalAdditionalLivingExpenses": "₹0.00", "TotalClaimAmount": "₹335,000.00" }
    ]
  },
  "out_PolicyDataJSON": {
    "insurer": { "name": "Apex Mutual Insurance Company", "address": "Home Office: Hartford, Connecticut" },
    "policy_form": { "form_type": "HO-3 Special Form", "form_code": "HO-3 (Rev. 01/2024)" },
    "declarations": {
      "policy_number": "HO-5534416561", "policy_type": "HO-3", "named_insured": "Rajesh Kumar Sharma",
      "mailing_address": { "line1": "12 MG Road, Apt 4B", "city": "Bengaluru", "state": "Karnataka", "pin": "560001" },
      "policy_period":   { "effective_date": "01/04/2025", "expiration_date": "31/03/2026" },
      "described_property": {
        "address": { "line1": "12 MG Road, Apt 4B", "city": "Bengaluru", "state": "Karnataka", "pin": "560001" },
        "property_type": "Apartment", "year_built": 2018, "square_footage": "1450 sq ft"
      },
      "annual_premium": "₹85,000.00", "payment_status": "Current - Paid in Full",
      "producing_agent": { "name": "Anita Iyer", "phone": "+91 80 1234 5678" }
    },
    "coverage_summary": {
      "currency": "INR",
      "coverages": [
        { "coverage_code": "A", "description": "Dwelling",                   "limit_of_liability": "₹4,000,000.00", "deductible": "₹15,000.00" },
        { "coverage_code": "B", "description": "Other Structures",           "limit_of_liability": "₹400,000.00",   "deductible": "₹15,000.00" },
        { "coverage_code": "C", "description": "Personal Property",          "limit_of_liability": "₹2,000,000.00", "deductible": "₹15,000.00" },
        { "coverage_code": "D", "description": "Loss of Use",                "limit_of_liability": "₹800,000.00",   "deductible": "None" },
        { "coverage_code": "E", "description": "Personal Liability",         "limit_of_liability": "₹1,500,000.00", "deductible": "None" },
        { "coverage_code": "F", "description": "Medical Payments to Others", "limit_of_liability": "₹30,000.00",    "deductible": "None" }
      ],
      "special_limits_of_liability_coverage_c": [
        { "property_category": "Jewelry, watches, furs, and precious stones",   "sublimit": "₹50,000.00" },
        { "property_category": "Electronics, computers, and related equipment", "sublimit": "₹100,000.00" }
      ]
    },
    "insuring_agreement": { "text": "In consideration of the premium paid, Apex Mutual Insurance Company agrees to provide insurance ..." },
    "section_I_covered_perils": {
      "coverage_A_and_B": { "type": "Open Perils", "description": "We insure against direct physical loss to property described in Coverages A and B from all causes of loss, except those losses specifically excluded ..." },
      "coverage_C":       { "type": "Named Perils", "perils": [ { "number": 1, "name": "Fire or Lightning" }, { "number": 2, "name": "Windstorm or Hail" }, { "number": 12, "name": "Accidental Discharge or Overflow of Water or Steam" } ] }
    },
    "section_II_exclusions": {
      "A_excluded_perils": [
        { "number": 1, "name": "Flood" }, { "number": 2, "name": "Earth Movement" }, { "number": 3, "name": "Water Damage" }
      ],
      "B_other_exclusions": [
        { "number": 10, "name": "Gradual Deterioration" }, { "number": 14, "name": "Faulty Construction" }
      ]
    },
    "section_III_special_conditions": [
      { "number": 1, "name": "Filing Deadline", "value": "60 calendar days" },
      { "number": 4, "name": "Vacancy Clause",  "value": "60 consecutive days; 15% reduction" }
    ],
    "section_IV_endorsements": [
      { "endorsement_number": 1, "name": "Replacement Cost Coverage", "status": "Included" }
    ],
    "section_V_general_conditions": [
      { "number": 1, "name": "Loss Settlement — Coverage A (Dwelling)" }
    ]
  }
}
```
