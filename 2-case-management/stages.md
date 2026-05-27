# Stages — implementation spec

Five primary stages and two secondary stages. For each, this file specifies the entry condition, the agents and robots invoked, the variable wiring (which case-entity fields feed which agent inputs), the persistence rules, the exit expressions, and per-stage timer / error handling.

Stage code (the value of `case.currentStage`) is the primary identifier. The pseudocode in this file uses a Maestro-friendly expression syntax: `case.<field>` reads from the case entity, `agent.<output>` reads from the most recent invocation in the current stage, `>=` and `==` are comparisons, `&&` and `||` are boolean operators.

---

## Primary stages

### Intake

| Aspect | Value |
|---|---|
| **Stage code** | `Intake` |
| **Type** | Primary |
| **Entry condition** | New case-entity row created by the Intake Robot when a claim packet arrives. |
| **Activities** | (1) Intake Robot reads incoming email or portal submission; (2) uploads `claimFormPdf`, `policyPdf` (always), and `assessmentReportPdf` (if present) to Storage Buckets; (3) creates the `PropertyClaimCase` row with header fields; (4) invokes IXP on the Claim Form PDF and writes `out_ClaimIXPDataJSON`; (5) looks up prior claims for the same `policyId` and `claimantName` (best effort) and writes `priorClaims[]`; (6) sends the acknowledgement email via Integration Service. |
| **Agents invoked** | None. |
| **Robots invoked** | `Intake Robot` |
| **Outputs persisted** | Header fields (claimId, policyId, claimantName, claimantEmail, claimantPhone, propertyAddress*, propertyCountry, incidentType, incidentDate, dateOfSubmission, totalClaimAmount), attachment refs (`claimFormPdf`, `policyPdf`, `assessmentReportPdf` if present), `out_ClaimIXPDataJSON`, `priorClaims[]`. |
| **Exit expression** | `case.claimFormPdf != null && case.policyPdf != null && case.out_ClaimIXPDataJSON != null` → transition to `EligibilityAnalysis`. Otherwise → `DetailsInquiry` (missing document path; v1 manual). |
| **Timer** | None. |
| **Error handling** | IXP confidence < 80% on any required field → Validation Station gate (case held at Intake until a human validates). Robot job failure → standard Orchestrator retry (3 attempts, 30s backoff). |

---

### Eligibility Analysis

| Aspect | Value |
|---|---|
| **Stage code** | `EligibilityAnalysis` |
| **Type** | Primary |
| **Entry condition** | Transition from `Intake` with all required inputs on the case. |
| **Activities** | Invoke the **01 Claim Eligibility Agent**. After the agent completes, branch on `out_isEligible` and `out_ClaimEligibilityJSON.recommendation`. |
| **Agents invoked** | `01_claim_eligibility` |
| **Robots invoked** | Email status notification (`Notification Robot`) on stage exit if outcome is `proceed_to_coverage_analysis`. |
| **Variable wiring (inputs)** | `in_ClaimIXPDataJSON ← case.out_ClaimIXPDataJSON`<br>`in_PolicyID ← case.policyId`<br>`in_PolicyPDF ← case.policyPdf` |
| **Outputs persisted** | `out_ClaimEligibilityJSON`, `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_isEligible`. Also set `case.currency` from `out_PolicyDataJSON.coverage_summary.currency`. |
| **Exit expression** | <pre>switch out_ClaimEligibilityJSON.recommendation:<br>  case "proceed_to_coverage_analysis":<br>    if case.assessmentReportPdf != null then → DataAnalysis<br>    else → AwaitingAssessment<br>  case "deny":<br>    → SettlementAndClosure (deny path)<br>  case "escalate":<br>    → ClaimReview (triggerStage = "eligibility")</pre> |
| **Timer** | None on the stage itself. Agent invocation has a 60s soft deadline; retry once on transient failure (per `CONFIG.md` §12). |
| **Error handling** | Agent persistent failure → synthesise a critical-failure envelope on the case and transition to `ClaimReview` for manual investigation. |

---

### Awaiting Assessment (Secondary)

| Aspect | Value |
|---|---|
| **Stage code** | `AwaitingAssessment` |
| **Type** | Secondary (conditional) |
| **Entry condition** | Eligibility Agent returned `proceed_to_coverage_analysis` AND `case.assessmentReportPdf` is null. |
| **Activities** | (1) `Assessor Intake Robot` sends an inspection-request email to the assessor panel via Integration Service; (2) the robot polls the assessor inbox / portal for the report PDF; (3) on receipt, the robot uploads the PDF to the `Assessor Reports` bucket and sets `case.assessmentReportPdf`. |
| **Agents invoked** | None. |
| **Robots invoked** | `Assessor Intake Robot` |
| **Variable wiring** | Reads `case.claimId`, `case.policyId`, `case.claimantName`, `case.propertyAddressLine1`, etc. for the email composition. |
| **Outputs persisted** | `assessmentReportPdf` attachment ref. |
| **Exit expression** | `case.assessmentReportPdf != null` → `DataAnalysis`. |
| **Timer** | 10 business days (configurable via `CONFIG.md`). On expiry, the robot emits a follow-up email and sets `case.reviewRequired = true` so the Process App surfaces the stuck case to operations. Stage does **not** auto-escalate; manual operator intervention is the v1 path. |
| **Error handling** | Email send failure → standard Integration Service retry policy. |

---

### Details Inquiry (Secondary)

| Aspect | Value |
|---|---|
| **Stage code** | `DetailsInquiry` |
| **Type** | Secondary (conditional) |
| **Entry condition** | Either: required document missing at Intake; OR a reviewer requests clarification from the Action App at Eligibility or Decision stages. The originating stage is recorded in `out_DetailsInquiryJSON.originating_stage`. |
| **Activities** | (1) `Inquiry Robot` composes and sends the request email; (2) Maestro enters a wait state with two timers: 7-day reminder and 14-day deadline; (3) on response received, robot writes `out_DetailsInquiryJSON.response_received_at` and `.response`; (4) Maestro resumes. |
| **Agents invoked** | None. |
| **Robots invoked** | `Inquiry Robot` |
| **Variable wiring** | Reads `case.claimantEmail`, the originating stage, and the requested-information list. |
| **Outputs persisted** | `out_DetailsInquiryJSON` (per the shape in `2-data-format/details-inquiry.md`). |
| **Exit expression** | <pre>switch out_DetailsInquiryJSON.status:<br>  case "responded":<br>    → out_DetailsInquiryJSON.originating_stage<br>  case "timed_out":<br>    → out_DetailsInquiryJSON.originating_stage (with case.reviewRequired = true so operations sees it)</pre> |
| **Timer** | 7-day reminder (sends a follow-up email), 14-day deadline (terminates the inquiry). |
| **Error handling** | Email send failure → retry. Multiple responses received → first response wins; later ones append to `case.priorClaims` or operator review queue. |
| **v1 note** | Automatic inquiry generation is out of scope for v1. The stage is reachable manually via the operations utility (see `testing-without-apps.md` §5). |

---

### Data Analysis

| Aspect | Value |
|---|---|
| **Stage code** | `DataAnalysis` |
| **Type** | Primary |
| **Entry condition** | Eligibility passed (directly or after reviewer-continue) AND `case.assessmentReportPdf != null`. |
| **Activities** | Three sub-steps in sequence: (1) **Validate**, (2) **Parallel analysis**, (3) **Synthesise**. |
| **Agents invoked** | `02_assessment_validation` → fork (`03_coverage_analysis`, `04_payout_calculation`, `05_credibility_assessment`) → join → `06_decision` |
| **Robots invoked** | `Notification Robot` on exit. |
| **Sub-step 1 — Validate** | Invoke **02 Assessment Validation Agent**.<br>Inputs: `in_ClaimDataJSON ← case.out_ClaimDataJSON`, `in_PolicyDataJSON ← case.out_PolicyDataJSON`, `in_AssessorReportPDF ← case.assessmentReportPdf`, `in_ClaimEligibilityJSON ← case.out_ClaimEligibilityJSON`, `in_EscalationDecision_Eligibility ← case.out_EscalationDecision_Eligibility (or null)`, `in_EscalationComment_Eligibility ← case.out_EscalationComment_Eligibility (or null)`.<br>Outputs: `out_AssessmentValidationJSON`, `out_AssessmentReportJSON`.<br>Branch: if `recommendation = "reject_report"` → set `case.reviewRequired = true` and transition to `ClaimReview` with `triggerStage = "decision"`. Otherwise continue to sub-step 2 (also if `recommendation = "escalate"` — those flags are aggregated by Agent 06). |
| **Sub-step 2 — Parallel** | Fork into three parallel branches and join when all three complete:<br>• **03 Coverage Analysis** — Inputs: `in_ClaimDataJSON`, `in_PolicyDataJSON`, `in_AssessmentReportJSON`, `in_ClaimEligibilityJSON`, `in_AssessmentValidationJSON`, `in_EscalationDecision_Eligibility`, `in_EscalationComment_Eligibility`. Output: `out_CoverageAnalysisJSON`.<br>• **04 Payout Calculation** — Same inputs plus the others. Output: `out_PayoutCalculationJSON`.<br>• **05 Credibility Assessment** — Same inputs plus `in_PriorClaimsJSON ← case.priorClaims`. Output: `out_CredibilityAssessmentJSON`. |
| **Sub-step 3 — Synthesise** | Invoke **06 Decision Agent**. Inputs are all five upstream envelopes plus the eligibility reviewer-notes pair. Outputs: `out_DecisionJSON`, `out_FinalDecision`. |
| **Outputs persisted** | All five envelopes plus `out_AssessmentReportJSON` and `out_FinalDecision`. |
| **Exit expression** | <pre>switch out_FinalDecision:<br>  case "approve", "partial_approve", "deny":<br>    → SettlementAndClosure<br>  case "escalate":<br>    → ClaimReview (triggerStage = "decision")</pre> |
| **Timer** | Each agent has a 60s soft deadline (120s for Agent 07, which is letter drafting in Stage 5). Retry once per agent on transient failure. |
| **Error handling** | If any of the three parallel agents fails persistently, synthesise a critical envelope for that agent and continue — Agent 06 will see the synthetic envelope and almost certainly emit `escalate`. The parallel block does not block on a single-agent failure. |

---

### Claim Review

| Aspect | Value |
|---|---|
| **Stage code** | `ClaimReview` |
| **Type** | Primary (conditional — only entered on escalation) |
| **Entry condition** | Eligibility Analysis or Data Analysis transitioned here because `recommendation = "escalate"` or `out_FinalDecision = "escalate"`. `case.triggerStage` is set to `"eligibility"` or `"decision"` respectively. |
| **Activities** | (1) Maestro creates an Action Center task with the input payload described in `action-center-tasks.md`; (2) Maestro enters a wait state until the task is completed; (3) the reviewer (or fake-reviewer script) submits `{ decision, reviewerNotes, reviewedAt }`; (4) Maestro maps the output to `case.out_EscalationDecision_<TriggerStage>`, `out_EscalationComment_<TriggerStage>`, `out_EscalationReviewedAt_<TriggerStage>`. |
| **Agents invoked** | None inside this stage. (Subsequent stages re-invoke agents that need the reviewer note.) |
| **Robots invoked** | None inside this stage. |
| **Exit expression** | <pre>let trig = case.triggerStage<br>let dec  = case["out_EscalationDecision_" + cap(trig)]<br><br>if trig == "eligibility":<br>  if dec == "continue":<br>    if case.assessmentReportPdf != null then → DataAnalysis<br>    else → AwaitingAssessment<br>  if dec == "deny":<br>    → SettlementAndClosure (deny path)<br><br>if trig == "decision":<br>  if dec == "continue":<br>    → SettlementAndClosure (use out_DecisionJSON.details.effective_decision_if_continued for the letter)<br>  if dec == "deny":<br>    → SettlementAndClosure (deny path — reviewer overrides)</pre> |
| **Timer** | 5 business days (configurable via `CONFIG.md`). On expiry → escalate to the Claims Operations Manager queue (out of scope for v1; in v1 the case sits in Action Center until completed). |
| **Error handling** | Action Center task creation failure → fall back to writing a synthetic "needs operator attention" entry on the case and notifying the ops queue. |

---

### Settlement and Closure

| Aspect | Value |
|---|---|
| **Stage code** | `SettlementAndClosure` |
| **Type** | Primary |
| **Entry condition** | Reached from `EligibilityAnalysis` (deny), `DataAnalysis` (auto-decision), or `ClaimReview` (post-review). |
| **Activities** | (1) Invoke **07 Claim Response Agent** to draft the decision letter; (2) `Settlement Robot` sends the letter email via Integration Service; (3) for approve / partial_approve outcomes, the robot posts a message to the `Finance Payout Queue`; (4) robot sets `case.status` to the final value and `case.closedAt`; (5) Maestro transitions to `Closed`. |
| **Agents invoked** | `07_claim_response` |
| **Robots invoked** | `Settlement Robot` |
| **Variable wiring (07 inputs)** | `in_ClaimDataJSON ← case.out_ClaimDataJSON`<br>`in_PolicyDataJSON ← case.out_PolicyDataJSON`<br>`in_ClaimEligibilityJSON ← case.out_ClaimEligibilityJSON`<br>`in_AssessmentValidationJSON ← case.out_AssessmentValidationJSON (or null)`<br>`in_CoverageAnalysisJSON ← case.out_CoverageAnalysisJSON (or null)`<br>`in_PayoutCalculationJSON ← case.out_PayoutCalculationJSON (or null)`<br>`in_CredibilityAssessmentJSON ← case.out_CredibilityAssessmentJSON (or null)`<br>`in_DecisionJSON ← case.out_DecisionJSON (or null on eligibility-deny path)`<br>`in_EscalationDecision_Eligibility ← case.out_EscalationDecision_Eligibility (or null)`<br>`in_EscalationComment_Eligibility ← case.out_EscalationComment_Eligibility (or null)`<br>`in_EscalationDecision_Decision ← case.out_EscalationDecision_Decision (or null)`<br>`in_EscalationComment_Decision ← case.out_EscalationComment_Decision (or null)` |
| **Outputs persisted** | `out_ClaimResponseJSON`, `letterEmailMessageId`, `paymentInstructionRef` (if applicable), `case.status`, `case.closedAt`. |
| **Exit expression** | Always → `Closed`. |
| **Settlement Robot rules** | <pre>let effective = out_ClaimResponseJSON.details.effective_decision<br><br>// 1. Send the letter email<br>let body = out_ClaimResponseJSON.details.letter_body<br>let to   = case.claimantEmail<br>let msg  = IntegrationService.Gmail.SendEmail(to, subject, body)<br>case.letterEmailMessageId = msg.id<br><br>// 2. Post payment instruction if applicable<br>if effective in ["approve", "partial_approve"] && out_ClaimResponseJSON.details.letter_metadata.net_payout_amount > 0:<br>  let pmt = build_payment_instruction(case, out_ClaimResponseJSON, out_PayoutCalculationJSON, out_DecisionJSON)<br>  Orchestrator.Queue("Finance Payout Queue").Enqueue(pmt)<br>  case.paymentInstructionRef = pmt.queueMessageId<br><br>// 3. Set status<br>switch effective:<br>  case "approve":          case.status = "approved"<br>  case "partial_approve":  case.status = "partial_approved"<br>  case "deny":             case.status = "denied"<br>  case "letter_pending_review": (should not occur at Settlement) raise error<br><br>case.closedAt = now()<br>case.currentStage = "Closed"</pre> |
| **Timer** | None. |
| **Error handling** | Email send failure → retry queue; case stays in SettlementAndClosure until dispatch confirmed. Finance Queue write failure → retry; case does not close until the message is on the queue. |

---

## Stage map (summary)

```
            ┌───────────┐
            │  Intake   │
            └─────┬─────┘
                  │ (claim form + policy present)
                  ▼
        ┌──────────────────────┐
        │  EligibilityAnalysis │── deny ────────────────────────────────────┐
        └──────┬───────────────┘                                            │
               │                                                            │
       ┌───────┼──────────┐                                                 │
       │       │          │                                                 │
   escalate    │       proceed_to_coverage_analysis                         │
       │       │          │                                                 │
       ▼       │          ▼                                                 │
  ┌─────────┐  │     ┌────────────────────┐  (no report yet)               │
  │ Claim   │  │     │ AwaitingAssessment │◄──────┐                        │
  │ Review  │  │     └─────────┬──────────┘       │                        │
  │ (elig)  │  │               │                  │                        │
  └────┬────┘  │               │ (report received)│                        │
       │       │               ▼                  │                        │
       │       │       ┌─────────────────┐        │                        │
       │       └──────►│  DataAnalysis    │       │                        │
       │ (continue)    └─────────┬───────┘        │                        │
       │                         │                │                        │
       │              ┌──────────┼──────────┐     │                        │
       │              │          │          │     │                        │
       │            auto      escalate   reject_report                     │
       │              │          │          │     │                        │
       │              │          ▼          ▼     │                        │
       │              │     ┌─────────┐    (review trigger)                │
       │              │     │ Claim   │                                    │
       │              │     │ Review  │                                    │
       │              │     │ (decn)  │                                    │
       │              │     └────┬────┘                                    │
       │              │          │                                         │
       │              │   (continue|deny)                                  │
       │              ▼          ▼                                         │
       └────────►┌─────────────────────┐ ◄──────────────────────────────────┘
                │ SettlementAndClosure │
                └──────────┬──────────┘
                           │
                           ▼
                       (Closed)

  Cross-cutting secondary stage:
   • DetailsInquiry can be entered from Intake, EligibilityAnalysis,
     or ClaimReview when an operator requests clarification. It
     returns to the originating stage on response or timeout.
```

---

*End of stages.md.*
