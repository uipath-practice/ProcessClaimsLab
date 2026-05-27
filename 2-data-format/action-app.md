# Action App contract (`claim-review`)

The Action App is rendered inside an Action Center task whenever Maestro creates one. It is the human reviewer's single-claim surface. The same app handles both escalation points (eligibility and decision) — the `triggerStage` input distinguishes them.

| Aspect | Value |
|---|---|
| App route name | `claim-review` |
| App type | Coded **Action App** |
| Trigger 1 | `triggerStage = "eligibility"` — created from Agent 01 escalation |
| Trigger 2 | `triggerStage = "decision"` — created from Agent 06 escalation |
| Authority | Fully open. The reviewer may continue or deny any claim regardless of agent recommendations. |
| Action App output cardinality | 2-way: `continue` or `deny`, with free-text `reviewerNotes`. |

---

## 1. Input payload (`action-schema.json`)

This is the payload Maestro hands to the Action Center task when it creates one. Every field is read-only inside the app.

```jsonc
{
  // ----------- routing -----------
  "claimId":      "CLM-2026-357861",
  "policyId":     "HO-5534416561",
  "triggerStage": "eligibility",                         // "eligibility" | "decision"

  // ----------- case header (for the summary cards) -----------
  "claimantName":      "Rajesh Kumar Sharma",
  "claimantEmail":     "rajesh.sharma@example.in",
  "currency":          "INR",
  "incidentType":      "Water Damage",
  "incidentDate":      "2026-05-20",                      // ISO 8601 (case-entity format)
  "dateOfSubmission":  "2026-05-26",
  "totalClaimAmount":  335000.00,

  // ----------- agent outputs available at trigger time -----------
  "out_ClaimDataJSON":           { /* ... */ },
  "out_PolicyDataJSON":          { /* ... */ },
  "out_AssessmentReportJSON":    { /* ... */ } | null,    // present only at decision trigger
  "out_ClaimEligibilityJSON":    { /* envelope */ },
  "out_AssessmentValidationJSON":{ /* envelope */ } | null,
  "out_CoverageAnalysisJSON":    { /* envelope */ } | null,
  "out_PayoutCalculationJSON":   { /* envelope */ } | null,
  "out_CredibilityAssessmentJSON": { /* envelope */ } | null,
  "out_DecisionJSON":            { /* envelope */ } | null,

  // ----------- attachments (Storage Bucket signed URLs) -----------
  "attachments": {
    "claimForm":         { "url": "https://...signed...", "fileName": "CLM-2026-357861-claim.pdf",        "mime": "application/pdf" },
    "policy":            { "url": "https://...signed...", "fileName": "HO-5534416561.pdf",                "mime": "application/pdf" },
    "assessmentReport":  { "url": "https://...signed...", "fileName": "CLM-2026-357861-incident.pdf",     "mime": "application/pdf" } | null
  },

  // ----------- prior claims reference data -----------
  "priorClaims": [
    {
      "claimId":          "CLM-2025-100234",
      "incidentDate":     "2025-08-12",
      "incidentType":     "Water Damage",
      "totalClaimAmount": 180000.00,
      "currency":         "INR",
      "finalDecision":    "approved",
      "closedAt":         "2025-09-04T13:42:00Z"
    }
  ]
}
```

### Trigger-time presence matrix

| Field | At `triggerStage="eligibility"` | At `triggerStage="decision"` |
|---|---|---|
| `out_ClaimDataJSON` | yes | yes |
| `out_PolicyDataJSON` | yes | yes |
| `out_AssessmentReportJSON` | null | yes |
| `out_ClaimEligibilityJSON` | yes | yes |
| `out_AssessmentValidationJSON` | null | yes |
| `out_CoverageAnalysisJSON` | null | yes |
| `out_PayoutCalculationJSON` | null | yes |
| `out_CredibilityAssessmentJSON` | null | yes |
| `out_DecisionJSON` | null | yes |
| `attachments.assessmentReport` | null | yes |

The Action App handles null values gracefully — panels for agents that haven't run yet are simply not rendered.

---

## 2. Output payload (Action App → Maestro)

What the Action App writes when the reviewer clicks **Continue** or **Deny**:

```jsonc
{
  "decision":      "continue",                  // "continue" | "deny"
  "reviewerNotes": "Late filing acceptable — claimant was hospitalised between 21/05 and 23/05; submission was made within 3 days of discharge.",
  "reviewedAt":    "2026-05-27T11:14:00Z"       // ISO 8601 timestamp with tz
}
```

Field rules:

| Field | Rules |
|---|---|
| `decision` | Required. Exactly one of `continue` or `deny`. |
| `reviewerNotes` | Required. Minimum 20 characters at the **eligibility** trigger; minimum 20 characters at the **decision** trigger. Empty is rejected by the form. |
| `reviewedAt` | Server-generated. The app submits the moment of click. |

---

## 3. How Maestro maps the output to case fields

Maestro writes the Action App output to four case-entity fields depending on which `triggerStage` was active:

### When `triggerStage = "eligibility"`

| Action App field | Case field |
|---|---|
| `decision` | `out_EscalationDecision_Eligibility` |
| `reviewerNotes` | `out_EscalationComment_Eligibility` |
| `reviewedAt` | `out_EscalationReviewedAt_Eligibility` |

### When `triggerStage = "decision"`

| Action App field | Case field |
|---|---|
| `decision` | `out_EscalationDecision_Decision` |
| `reviewerNotes` | `out_EscalationComment_Decision` |
| `reviewedAt` | `out_EscalationReviewedAt_Decision` |

---

## 4. Routing after the reviewer submits

Maestro routes based on the combination of `triggerStage` and `decision`:

| Trigger | Decision | Next stage |
|---|---|---|
| `eligibility` | `continue` | `DataAnalysis` (or `AwaitingAssessment` if the assessor report is still missing) |
| `eligibility` | `deny` | `SettlementAndClosure` (denial letter) |
| `decision` | `continue` | `SettlementAndClosure` (effective decision is then derived from `out_DecisionJSON` — typically the agents' recommended approve/partial; the reviewer "continued" past the escalation) |
| `decision` | `deny` | `SettlementAndClosure` (denial letter — reviewer overrides any approval) |

The reviewer's authority is unbounded at both trigger stages.

---

## 5. `reviewerNotes` propagation downstream

After an eligibility-stage override, the reviewer's `reviewerNotes` (`out_EscalationComment_Eligibility`) is propagated as input to every downstream agent: 02, 03, 04, 05, 06, and 07. Downstream agents must treat the reviewer's decision as authoritative and must not re-raise the concern that the reviewer has resolved.

Agent prompts include the directive (CONFIG.md §10):

> *If a reviewer override is present in your inputs, treat the reviewer decision as final. Do not re-raise concerns that the reviewer addressed in `reviewerNotes`. Use those notes to inform your own reasoning where they are relevant.*

After a decision-stage override, the override goes only to Agent 07 (the only downstream agent), which incorporates it into the letter.

---

## 6. UI components used by the Action App

(From SDD §8.3 — listed here for completeness.)

| Component | Role inside the Action App |
|---|---|
| `TabbedAnalysis` | One tab per agent output present in the input payload. Status badge per tab from envelope `status`. |
| `GenericAnalysisPanel` | Renders any envelope through one component. |
| `AgentDetailsPanel` | Pluggable inner panel for the `details` block of each agent (registered by slug). Falls back to a JSON tree. |
| `SummaryBlocks` | Three top cards: Claim, Policy, Assessment summary, each with a "Preview PDF" button → opens the signed URL. |
| `AssessmentValidationPanel` | Renders Agent 02's `details` (assessment report validation findings). |
| `ClaimResponsePanel` | Renders Agent 07's draft letter at `decision` trigger if the agents already produced one. Read-only preview. |
| `DecisionActionBar` | Two-stage Continue / Deny confirmation. Disabled until `reviewerNotes` reaches the minimum length. |
| `LifecycleBar` | Visualises stages 1–5 + secondary stages; highlights the current stage. |

---

## 7. Failure modes

| Scenario | App behaviour |
|---|---|
| One or more `out_<agent>JSON` fields are null at decision trigger | Panel is omitted from the tab bar with a "Not yet available" placeholder. |
| Signed URL has expired before the reviewer opens the PDF | "Preview unavailable" message; the app refreshes the signed URL via the OAuth-backed API and retries. |
| Reviewer submits without `reviewerNotes` | Form rejects with a min-length error before the API call. |
| Reviewer submits but the Action Center task has already been completed by another session | The submit API returns 409; the app shows "Task already completed" and routes back to the task list. |

---

*End of action-app.md.*
