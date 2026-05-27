# Document-derived data structures

This file defines the three JSON shapes derived from the input documents. They are the inputs to almost every downstream agent.

| Variable | Source | Producer | Consumed by |
|---|---|---|---|
| `out_ClaimDataJSON` | Claim Form (FNOL) PDF + IXP extraction | Agent 01 (cleans the IXP output) | Agents 02, 03, 04, 05, 06, 07 |
| `out_PolicyDataJSON` | Insurance Policy PDF | Agent 01 (reads the PDF directly — see CONFIG §8) | Agents 02, 03, 04, 05, 06, 07 |
| `out_AssessmentReportJSON` | Professional Incident Report PDF | Agent 02 (reads the PDF directly) | Agents 03, 04, 05, 06, 07 |

Each shape is **canonical** — the field names and structure below are what every downstream agent expects to find.

---

## 1. `out_ClaimDataJSON`

Cleaned representation of the FNOL claim form. Confidence scores and OCR metadata are stripped from the IXP output. The structure mirrors the prior iteration so existing test data continues to work.

Source template: `inputs/document samples/Templates/property-claims/claim-submission-form/specification.md`.

```jsonc
{
  "Claim": [
    {
      "ClaimID":          "CLM-2026-357861",
      "InsurerName":      "Apex Mutual Insurance Company",       // extracted from the claim-form header (branding block)
      "InsurerAddress":   "Claims Department, P.O. Box 9400, Hartford, CT 06152",
      "DateOfSubmission": "26/05/2026"                            // DD/MM/YYYY
    }
  ],

  "ClaimClaimant": [
    {
      "Name":         "Rajesh Kumar Sharma",
      "PhoneNumber":  "+91 98765 43210",
      "EmailAddress": "rajesh.sharma@example.in",
      "PolicyNumber": "HO-5534416561"
    }
  ],

  "ClaimProperty": [
    {
      "StreetAddress": "12 MG Road, Apt 4B",
      "City":          "Bengaluru",
      "State":         "Karnataka",
      "ZipCode":       "560001",
      "IsPrimary":     true,
      "IsPresent":     true
    }
  ],

  "ClaimIncident": [
    {
      "DateOfIncident":            "20/05/2026",                  // DD/MM/YYYY
      "TimeOfIncident":            "approximately 22:30",
      "TypeOfIncident":            "Water Damage",
      "DescriptionOfIncident":     "Burst pipe in kitchen flooded ground floor.",
      "PoliceReportFiled":         false,
      "PoliceReportNumber":        null,
      "EmergencyServicesCalled":   false,
      "TemporaryRepairsMade":      true,
      "TemporaryRepairsDescription": "Main water shutoff closed; affected area dried with fans for 24h."
    }
  ],

  "ClaimDamageInventory": [
    {
      "Category":        "Structure - Flooring",
      "Location":        "Kitchen",
      "Description":     "Engineered timber flooring warped from water exposure, 25 sqm affected",
      "EstimatedCost":   "₹125,000.00",                            // formatted string with symbol — keep as-found
      "RepairOrReplace": "Replace"
    }
    // ... up to 5 items per the template (Section 4 has 5 rows)
  ],

  "ClaimClaimTotals": [
    {
      "TotalStructureDamage":        "₹275,000.00",
      "TotalPersonalProperty":       "₹60,000.00",
      "TotalAdditionalLivingExpenses": "₹0.00",
      "TotalClaimAmount":            "₹335,000.00"
    }
  ]
}
```

### Field notes

| Field | Notes |
|---|---|
| `Claim[0].InsurerName` and `InsurerAddress` | Not explicit `{{TOKEN}}` slots in the template — extracted from the document **header / branding block**. Agents read these from the textual header. |
| `Claim[0].DateOfSubmission` | Template field is `{{CLAIM_DATE}}`. Agents emit `DD/MM/YYYY`. Edge-case: if the source PDF carries MM/DD/YYYY ambiguity, Agent 01 must flag the ambiguity in its E-5 check rather than guess. |
| `ClaimProperty[0].IsPrimary` and `IsPresent` | Template has Yes/No fields. Agents convert to boolean. |
| `ClaimDamageInventory` | Up to 5 items in the template (5 rows). Agents preserve all items present. Empty rows are skipped — not emitted as items. |
| `ClaimDamageInventory[].EstimatedCost` and totals | **Kept as formatted strings with currency symbol**, NOT normalised to bare numbers. This preserves provenance for human review. Downstream agents that need to do arithmetic strip the symbol per `normalisation.md`. |
| Currency throughout | Inferred from the policy currency (`out_PolicyDataJSON.coverage_summary.currency`), not parsed here. |
| Locale | Not stored in this shape. Derived from the property address country when needed. |

---

## 2. `out_PolicyDataJSON`

Structured representation of the HO-3 Insurance Policy PDF.

Source template: `inputs/document samples/Templates/property-claims/insurance-policy/specification.md`.

```jsonc
{
  "insurer": {
    "name":    "Apex Mutual Insurance Company",
    "address": "Home Office: Hartford, Connecticut"
  },

  "policy_form": {
    "form_type": "HO-3 Special Form",
    "form_code": "HO-3 (Rev. 01/2024)"
  },

  "declarations": {
    "policy_number":    "HO-5534416561",
    "policy_type":      "HO-3",
    "named_insured":    "Rajesh Kumar Sharma",
    "mailing_address":  { "line1": "12 MG Road, Apt 4B", "city": "Bengaluru", "state": "Karnataka", "pin": "560001" },
    "policy_period":    { "effective_date": "01/04/2026", "expiration_date": "31/03/2027" },  // DD/MM/YYYY
    "described_property": {
      "address":        { "line1": "12 MG Road, Apt 4B", "city": "Bengaluru", "state": "Karnataka", "pin": "560001" },
      "property_type":  "Apartment",
      "year_built":     2018,
      "square_footage": "1450 sq ft"
    },
    "annual_premium":   "₹85,000.00",
    "payment_status":   "Current - Paid in Full",
    "producing_agent":  { "name": "Anita Iyer", "phone": "+91 80 1234 5678" }
  },

  "coverage_summary": {
    "currency": "INR",                                       // ISO 4217 — agents normalise from the policy symbol
    "coverages": [
      { "coverage_code": "A", "description": "Dwelling",                       "limit_of_liability": "₹4,000,000.00", "deductible": "₹15,000.00" },
      { "coverage_code": "B", "description": "Other Structures",               "limit_of_liability": "₹400,000.00",   "deductible": "₹15,000.00" },
      { "coverage_code": "C", "description": "Personal Property",              "limit_of_liability": "₹2,000,000.00", "deductible": "₹15,000.00" },
      { "coverage_code": "D", "description": "Loss of Use",                    "limit_of_liability": "₹800,000.00",   "deductible": "None" },
      { "coverage_code": "E", "description": "Personal Liability",             "limit_of_liability": "₹1,500,000.00", "deductible": "None" },
      { "coverage_code": "F", "description": "Medical Payments to Others",     "limit_of_liability": "₹30,000.00",    "deductible": "None" }
    ],
    "special_limits_of_liability_coverage_c": [
      { "property_category": "Jewelry, watches, furs, and precious stones",  "sublimit": "₹50,000.00" },
      { "property_category": "Electronics, computers, and related equipment", "sublimit": "₹100,000.00" },
      { "property_category": "Cash, bank notes, bullion, and coins",          "sublimit": "₹4,000.00" }
      // ... preserve every category that appears in the policy
    ]
  },

  "insuring_agreement": {
    "text": "In consideration of the premium paid, Apex Mutual Insurance Company agrees to provide insurance ..."
  },

  "section_I_covered_perils": {
    "coverage_A_and_B": {
      "type":        "Open Perils",
      "description": "We insure against direct physical loss to property described in Coverages A and B from all causes of loss, except those losses specifically excluded ..."
    },
    "coverage_C": {
      "type":   "Named Perils",
      "perils": [
        { "number": 1,  "name": "Fire or Lightning" },
        { "number": 2,  "name": "Windstorm or Hail" },
        { "number": 3,  "name": "Explosion" },
        { "number": 4,  "name": "Riot or Civil Commotion" },
        { "number": 5,  "name": "Aircraft" },
        { "number": 6,  "name": "Vehicles" },
        { "number": 7,  "name": "Smoke" },
        { "number": 8,  "name": "Vandalism or Malicious Mischief" },
        { "number": 9,  "name": "Theft" },
        { "number": 10, "name": "Falling Objects" },
        { "number": 11, "name": "Weight of Ice, Snow, or Sleet" },
        { "number": 12, "name": "Accidental Discharge or Overflow of Water or Steam" },
        { "number": 13, "name": "Tearing Apart, Cracking, Burning, or Bulging" },
        { "number": 14, "name": "Freezing" },
        { "number": 15, "name": "Sudden and Accidental Damage from Artificially Generated Electrical Current" },
        { "number": 16, "name": "Volcanic Eruption" }
      ]
    }
  },

  "section_II_exclusions": {
    "A_excluded_perils": [
      { "number": 1, "name": "Flood" },
      { "number": 2, "name": "Earth Movement" },
      { "number": 3, "name": "Water Damage" },
      { "number": 4, "name": "Power Failure" },
      { "number": 5, "name": "Neglect" },
      { "number": 6, "name": "War" },
      { "number": 7, "name": "Nuclear Hazard" },
      { "number": 8, "name": "Intentional Loss" },
      { "number": 9, "name": "Governmental Action" }
    ],
    "B_other_exclusions": [
      { "number": 10, "name": "Gradual Deterioration" },
      { "number": 11, "name": "Pest Infestation" },
      { "number": 12, "name": "Settling, Cracking, Shrinking, Bulging, or Expansion" },
      { "number": 13, "name": "Smog" },
      { "number": 14, "name": "Faulty Construction" }
    ]
  },

  "section_III_special_conditions": [
    { "number": 1, "name": "Filing Deadline",      "value": "60 calendar days" },
    { "number": 2, "name": "Proof of Loss",        "value": "90 calendar days" },
    { "number": 3, "name": "Duty to Mitigate",     "value": "Insured must take reasonable steps to prevent further damage." },
    { "number": 4, "name": "Vacancy Clause",       "value": "60 consecutive days; 15% reduction; specified exclusions" },
    { "number": 5, "name": "Examination Under Oath" },
    { "number": 6, "name": "Other Insurance",      "value": "Pro rata" }
  ],

  "section_IV_endorsements": [
    { "endorsement_number": 1, "name": "Replacement Cost Coverage", "status": "Included" },   // or "Not Included"
    { "endorsement_number": 2, "name": "Ordinance or Law Coverage",  "status": "Not Included" }
  ],

  "section_V_general_conditions": [
    { "number": 1, "name": "Loss Settlement — Coverage A (Dwelling)" },
    { "number": 2, "name": "Loss Settlement — Coverage C (Personal Property)" },
    { "number": 3, "name": "Abandonment" },
    { "number": 4, "name": "Appraisal" },
    { "number": 5, "name": "Mortgage Clause" },
    { "number": 6, "name": "Cancellation" },
    { "number": 7, "name": "Liberalization" }
  ]
}
```

### Field notes

| Field | Notes |
|---|---|
| `coverage_summary.currency` | Always ISO 4217 (`USD`, `INR`, `HKD`, `EUR`, `GBP`, ...). Agent 01 derives this from the policy's currency symbols and surrounding context. |
| `coverage_summary.coverages[].limit_of_liability` and `deductible` | Kept as formatted strings with the source symbol. Downstream arithmetic strips the symbol per `normalisation.md`. |
| `special_limits_of_liability_coverage_c[]` | Preserve **all** sublimits present in the source policy — both the variable ones (`{{JEWELRY_SUBLIMIT}}`, `{{ELECTRONICS_SUBLIMIT}}`) and the hard-coded ones (cash, securities, firearms, etc.). Don't assume defaults. |
| `section_IV_endorsements` | The Replacement Cost Endorsement (#1) drives settlement basis (RC vs ACV). If `status = "Included"`, Agent 04 settles at replacement cost. Otherwise ACV (see PDD §12.6). |
| `section_III_special_conditions` | The Filing Deadline (60 days), Vacancy Clause (60-day threshold + 15% reduction), and Duty to Mitigate values are concrete contract numbers — preserve them. |

---

## 3. `out_AssessmentReportJSON`

Structured representation of the Professional Incident Report PDF.

Source template: `inputs/document samples/Templates/property-claims/incident-report/specification.md`.

```jsonc
{
  "report_metadata": {
    "report_reference": "RPT-CLM-2026-357861",
    "assessor": {
      "name":    "Vikram Patel",
      "license": "IRDAI-LP-12345",
      "company": "Patel & Associates Surveyors Pvt. Ltd."
    },
    "assessment_date":   "23/05/2026"            // DD/MM/YYYY
  },

  "property": {
    "address": {
      "line1":   "12 MG Road, Apt 4B",
      "city":    "Bengaluru",
      "state":   "Karnataka",
      "zip":     "560001"
    },
    "policyholder_name": "Rajesh Kumar Sharma",
    "claim_reference":   "CLM-2026-357861",
    "incident_date":     "20/05/2026",
    "reported_incident_type": "Water Damage"
  },

  "findings": {
    "narrative":        "On-site inspection conducted on 23/05/2026 confirmed extensive water damage to ground-floor kitchen, dining room, and adjacent passage flooring ...",
    "cause_determination":      "Failure of a 3/4-inch copper feeder pipe behind the kitchen sink cabinet, consistent with reported burst pipe incident.",
    "damage_scope_description": "Engineered timber flooring throughout 25 sqm of kitchen and 12 sqm of dining room is warped and unsalvageable. Drywall to a height of 30 cm above floor level is saturated. Kitchen base cabinets are water-damaged ..."
  },

  "repair_schedule": [
    { "description": "Remove and replace engineered timber flooring (37 sqm)",        "estimated_cost": "₹110,000.00" },
    { "description": "Drywall replacement to 30 cm above floor level (perimeter)",     "estimated_cost": "₹25,000.00" },
    { "description": "Kitchen base cabinet replacement (3 units)",                     "estimated_cost": "₹75,000.00" },
    { "description": "Plumbing repair — replace failed copper feeder section",         "estimated_cost": "₹15,000.00" },
    { "description": "Industrial drying and dehumidification (5 days)",                "estimated_cost": "₹30,000.00" }
  ],

  "cost_summary": {
    "total_estimated_repair_cost": "₹255,000.00"
  },

  "assessor_opinion": "The damage observed is consistent in scope and pattern with the reported burst pipe event on 20/05/2026. The claimant took reasonable mitigating steps. The estimated repair cost is in line with current Bengaluru market rates."
}
```

### Field notes

| Field | Notes |
|---|---|
| `report_metadata.assessor.license` | Required for AV-3 (assessor credentials check). When missing, AV-3 fails. |
| `property.claim_reference` and `property.policyholder_name` | Used by AV-4 (property / claim match). Must match the corresponding fields in `out_ClaimDataJSON` and `out_PolicyDataJSON`. |
| `findings.cause_determination` | The authoritative peril for Agent 03 coverage analysis when the claimant's narrative and the assessor's findings conflict. |
| `repair_schedule[]` | All line items present in the source report, in order. Used by Agent 04 for the reasonableness ratio (CR-2 / P-8). |
| `cost_summary.total_estimated_repair_cost` | The single number Agent 04 compares against the claimant's `TotalClaimAmount` for the ratio check. Currency must be the same as the policy currency — agents flag if it differs. |

---

## 4. Other generator-only metadata

The synthetic-document generator emits a manifest alongside each claim packet (e.g. `inputs/document samples/CLM-2026-357861-claim.json`):

```jsonc
{
  "ProfileId":            "claims-profile-11",
  "ProfileName":          "Rajesh Kumar Sharma",
  "Country":              "IN",
  "Currency":             "₹",                              // raw symbol
  "Difficulty":           "easy",
  "Documents":            [ { "DocumentType": "claim-submission-form", "FilePath": "...", ... } ],
  "AppliedDiscrepancies": [ "CLAIMS_LATE_FILING" ],              // test verification only
  "GeneratedAt":          "2026-05-26 05:19:22",
  "ClaimId":              "CLM-2026-357861",
  "PolicyId":             "HO-5534416561"
}
```

**Agents never read this manifest.** It exists for the test harness: after a generated claim is processed, the harness compares `AppliedDiscrepancies` against the rule IDs the agents fired to verify they caught the intended issues.

---

*End of document-fields.md.*
