# `PropertyClaimCase` — Data Service / Data Fabric entity

This is the persistent record for a claim. Exactly one row per claim. Every component (agents, robots, apps) reads from and writes to this entity. The entity is the **system of record** — no shared state lives anywhere else.

The entity is enriched stage by stage. Most fields are nullable because a case starts at Intake with only header data and accumulates outputs as the pipeline runs.

---

## 1. Field catalogue

| Group | Field | Type | Required at intake? | Source / Producer |
|---|---|---|---|---|
| **Identity** | `claimId` | string (key) | yes | Intake robot — from `out_ClaimIXPDataJSON.Claim[0].ClaimID` |
| | `policyId` | string | yes | Intake robot — from `out_ClaimIXPDataJSON.ClaimClaimant[0].PolicyNumber` |
| **Header (denormalised, populated at intake)** | `claimantName` | string | yes | Intake robot — from `out_ClaimIXPDataJSON.ClaimClaimant[0].Name` |
| | `claimantEmail` | string | yes | Intake robot |
| | `claimantPhone` | string | no | Intake robot |
| | `propertyAddressLine1` | string | yes | Intake robot |
| | `propertyCity` | string | yes | Intake robot |
| | `propertyState` | string | yes | Intake robot |
| | `propertyZip` | string | no | Intake robot |
| | `propertyCountry` | string (ISO 3166-1 alpha-2) | yes | Intake robot — derived from address |
| | `currency` | string (ISO 4217) | filled by Agent 01 | Agent 01 — normalised from policy |
| | `incidentType` | string | yes | Intake robot — from `out_ClaimIXPDataJSON.ClaimIncident[0].TypeOfIncident` |
| | `incidentDate` | date (ISO 8601 `YYYY-MM-DD`) | yes | Intake robot — normalised from `DD/MM/YYYY` |
| | `dateOfSubmission` | date (ISO 8601) | yes | Intake robot — normalised from `DD/MM/YYYY` |
| | `totalClaimAmount` | decimal | yes | Intake robot — bare number, currency from `currency` field |
| **Lifecycle** | `currentStage` | enum | yes | Maestro |
| | `status` | enum | yes | Maestro / Agent 06 / Settlement robot |
| | `triggerStage` | enum: `eligibility` \| `decision` \| null | conditional | Maestro — set when an Action Center task is created |
| | `reviewRequired` | boolean | yes | Maestro |
| | `createdAt` | timestamp (ISO 8601 with tz) | yes | Intake robot |
| | `updatedAt` | timestamp (ISO 8601 with tz) | yes | Every write |
| | `closedAt` | timestamp (ISO 8601 with tz) | conditional | Settlement robot |
| **Attachments (Storage Bucket references)** | `claimFormPdf` | attachment-ref | yes | Intake robot |
| | `policyPdf` | attachment-ref | yes | Intake robot |
| | `assessmentReportPdf` | attachment-ref | conditional | Assessor-intake robot (filled during Awaiting Assessment) |
| **Reference data** | `priorClaims` | array of `PriorClaim` (see §3) | conditional | Intake robot — best-effort lookup |
| **Intake outputs** | `out_ClaimIXPDataJSON` | object | yes | IXP |
| **Agent 01 outputs** | `out_ClaimEligibilityJSON` | envelope object | filled at Stage 2 | Agent 01 |
| | `out_ClaimDataJSON` | object | filled at Stage 2 | Agent 01 |
| | `out_PolicyDataJSON` | object | filled at Stage 2 | Agent 01 |
| | `out_isEligible` | boolean | filled at Stage 2 | Agent 01 (routing scalar) |
| **Agent 02 outputs** | `out_AssessmentValidationJSON` | envelope object | filled at Stage 3 entry | Agent 02 |
| | `out_AssessmentReportJSON` | object | filled at Stage 3 entry | Agent 02 |
| **Agent 03 output** | `out_CoverageAnalysisJSON` | envelope object | filled at Stage 3 parallel | Agent 03 |
| **Agent 04 output** | `out_PayoutCalculationJSON` | envelope object | filled at Stage 3 parallel | Agent 04 |
| **Agent 05 output** | `out_CredibilityAssessmentJSON` | envelope object | filled at Stage 3 parallel | Agent 05 |
| **Agent 06 outputs** | `out_DecisionJSON` | envelope object | filled at Stage 3 synthesis | Agent 06 |
| | `out_FinalDecision` | enum: `approve` \| `partial_approve` \| `deny` \| `escalate` | filled at Stage 3 synthesis | Agent 06 (routing scalar) |
| **Agent 07 output** | `out_ClaimResponseJSON` | envelope object | filled at Stage 5 | Agent 07 |
| **Human review** | `out_EscalationDecision_Eligibility` | enum: `continue` \| `deny` | conditional | Action App (eligibility task) |
| | `out_EscalationComment_Eligibility` | string | conditional | Action App (eligibility task) |
| | `out_EscalationReviewedAt_Eligibility` | timestamp | conditional | Action App (eligibility task) |
| | `out_EscalationDecision_Decision` | enum: `continue` \| `deny` | conditional | Action App (decision task) |
| | `out_EscalationComment_Decision` | string | conditional | Action App (decision task) |
| | `out_EscalationReviewedAt_Decision` | timestamp | conditional | Action App (decision task) |
| **Inquiry (secondary stage)** | `out_DetailsInquiryJSON` | object (see `details-inquiry.md`) | conditional | Inquiry robot |
| **Settlement** | `paymentInstructionRef` | string | conditional | Settlement robot — Finance Queue message ID |
| | `letterEmailMessageId` | string | conditional | Settlement robot — Integration Service email ID |

---

## 2. Enum values

### `currentStage`
```
Intake | EligibilityAnalysis | AwaitingAssessment | DetailsInquiry | DataAnalysis | ClaimReview | SettlementAndClosure | Closed
```

### `status`
```
pending_review        // Intake / between stages
in_progress           // Any agent is currently running
awaiting_assessment   // Awaiting Assessment secondary stage
awaiting_details      // Details Inquiry secondary stage
escalated             // Claim Review primary stage (Action Center task open)
approved              // Settlement complete with full approval
partial_approved      // Settlement complete with partial approval
denied                // Settlement complete with denial
closed                // Archival complete
```

### `triggerStage`
```
eligibility    // Action Center task created from Eligibility Analysis escalation
decision       // Action Center task created from Data Analysis (Decision Agent) escalation
null           // No open Action Center task
```

---

## 3. Embedded shapes

### `attachment-ref`

```jsonc
{
  "ID":            "5f9040d3-e938-4671-4add-08de9ae210e2",   // Orchestrator attachment key
  "FullName":      "HO-5534416561.pdf",
  "MimeType":      "application/pdf",
  "BucketName":    "Insurance Policies Repository",
  "StoragePath":   "claims/CLM-2026-357861/HO-5534416561.pdf"
}
```

### `PriorClaim` (best-effort enrichment)

```jsonc
{
  "claimId":          "CLM-2025-100234",
  "incidentDate":     "12/08/2025",                  // DD/MM/YYYY
  "incidentType":     "Water Damage",
  "totalClaimAmount": 180000.00,
  "currency":         "INR",
  "finalDecision":    "approved",                    // approve | partial_approve | deny
  "closedAt":         "2025-09-04T13:42:00Z"
}
```

Up to 10 most recent prior claims for the same `policyId` or `claimantName`. Best-effort lookup at intake — if the lookup fails, this is an empty array.

---

## 4. Field naming convention on the case entity

UiPath Data Service entities use **PascalCase or camelCase**. This lab uses:

- Header / domain fields: **camelCase** (`claimId`, `currentStage`, `claimantEmail`).
- Agent outputs and reviewer fields: **`out_<...>` prefix preserved** to mirror the Maestro variable name and make traceability obvious in the case record viewer.

This is the only mixed-case convention in the project. It's deliberate: when a developer opens the case entity record viewer, every `out_*` field maps to exactly one Maestro variable and one agent.

---

## 5. Validation rules at the entity level

Beyond field-level types, the entity enforces:

| Rule | Description |
|---|---|
| **R-CE-1** | `claimId` is unique. |
| **R-CE-2** | `incidentDate` ≤ `dateOfSubmission`. (Hard rule — guards against the sample-claim date-format issue.) |
| **R-CE-3** | If `currentStage = ClaimReview`, then `triggerStage` is one of `eligibility` or `decision`. |
| **R-CE-4** | If `out_FinalDecision = approve` then `out_DecisionJSON.details.approved_payout_reference.net_payout > 0`. |
| **R-CE-5** | `closedAt` is set if and only if `currentStage = Closed`. |
| **R-CE-6** | `currency` matches `out_PolicyDataJSON.coverage_summary.currency` once both are set. |

Failures of these rules are surfaced to the Process App as data-quality flags and to Maestro as case-event alerts.

---

## 6. Lifecycle transition matrix

(Repeated from SDD §7.5 for self-containment; the SDD is canonical.)

| From | To | Trigger |
|---|---|---|
| `Intake` | `EligibilityAnalysis` | All three documents (or at least Claim Form and Policy) classified and extracted |
| `Intake` | `DetailsInquiry` | Required document missing |
| `EligibilityAnalysis` | `AwaitingAssessment` | Eligibility pass and assessor report not yet on file |
| `EligibilityAnalysis` | `DataAnalysis` | Eligibility pass and assessor report present |
| `EligibilityAnalysis` | `SettlementAndClosure` | Eligibility hard deny |
| `EligibilityAnalysis` | `ClaimReview` | Eligibility recommendation = `escalate` |
| `EligibilityAnalysis` | `DetailsInquiry` | Reviewer requests clarification (from Action App) |
| `AwaitingAssessment` | `DataAnalysis` | Assessor report received |
| `DataAnalysis` | `SettlementAndClosure` | `out_FinalDecision` ∈ {approve, partial_approve, deny} |
| `DataAnalysis` | `ClaimReview` | `out_FinalDecision = escalate` |
| `ClaimReview` | `SettlementAndClosure` | Reviewer outcome recorded |
| `ClaimReview` | `DetailsInquiry` | Reviewer requests additional information |
| `DetailsInquiry` | (originating stage) | Claimant response received or 14-day timer expires |
| `SettlementAndClosure` | `Closed` | Letter sent, payment instruction issued, archive complete |

---

*End of case-entity.md.*
