# Maestro process — orchestration

This file defines the Maestro process layout: top-level state machine, variable scopes, expression syntax conventions, the parallel block at Data Analysis, the wait states, and the routing scalars.

A coding agent generating the Maestro process should produce one process definition that follows this structure. Each named "block" below maps to a Maestro activity or composite activity.

---

## 1. Process identity

| Property | Value |
|---|---|
| Process name | `PropertyClaimAdjudication` |
| Version | semver, set at deployment time |
| Trigger | Intake Robot creates the `PropertyClaimCase` row; Maestro starts the process on the new row. |
| Case entity | `PropertyClaimCase` (one process instance per row) |
| Termination | Reaches `Closed` stage |

---

## 2. Variable scope

Maestro variables live in two scopes:

| Scope | Purpose | Examples |
|---|---|---|
| **Process** (lives for the whole case) | All agent envelopes, routing scalars, case header fields, reviewer outputs. | `out_ClaimEligibilityJSON`, `out_FinalDecision`, `out_EscalationDecision_Eligibility` |
| **Block-local** (lives for the duration of one sequential or parallel block) | Temporary expressions, loop counters, attempt counts. | `attempt_count`, `agent_invocation_error` |

Every process-scope variable maps 1:1 to a field on the case entity. Maestro writes back to the case after each significant step so that the case row always reflects the live state.

---

## 3. Expression syntax conventions

The pseudocode below uses a Maestro-flavoured expression syntax that maps to Studio activities:

| Expression form | Meaning |
|---|---|
| `case.<field>` | Read from the case entity (Data Service query keyed on `case.claimId`). |
| `process.<varName>` | Read from the process-scope variable. |
| `agent.<agentSlug>.<output>` | Read the output of the most recent invocation of that agent in the current process run. |
| `task.<output>` | Read the output of the most recent Action Center task in the current process run. |
| `?:` | Ternary. `a ?: b` returns `a` if non-null, else `b`. |

Each "set case.X = Y" pseudo-statement maps to a Data Service Update Row activity in Maestro.

---

## 4. Top-level process layout

```
[ STAGE: Intake — handled by the Intake Robot, not Maestro ]
                          │
                          ▼
   ┌──────────────────────────────────────────────────────────┐
   │ STAGE: EligibilityAnalysis                               │
   │                                                          │
   │   set case.currentStage = "EligibilityAnalysis"          │
   │                                                          │
   │   invoke 01_claim_eligibility with                       │
   │     in_ClaimIXPDataJSON = case.out_ClaimIXPDataJSON      │
   │     in_PolicyID         = case.policyId                  │
   │     in_PolicyPDF        = case.policyPdf                 │
   │                                                          │
   │   set case.out_ClaimEligibilityJSON = agent.01_*.out_ClaimEligibilityJSON
   │   set case.out_ClaimDataJSON        = agent.01_*.out_ClaimDataJSON
   │   set case.out_PolicyDataJSON       = agent.01_*.out_PolicyDataJSON
   │   set case.out_isEligible           = agent.01_*.out_isEligible
   │   set case.currency                 = case.out_PolicyDataJSON.coverage_summary.currency
   │                                                          │
   │   Notification Robot job(case.claimantEmail, "Eligibility outcome")
   │                                                          │
   │   branch on case.out_ClaimEligibilityJSON.recommendation:│
   │     "proceed_to_coverage_analysis":                      │
   │       if case.assessmentReportPdf != null                │
   │         → DataAnalysis                                   │
   │       else                                               │
   │         → AwaitingAssessment                             │
   │     "deny":                                              │
   │       set case.out_FinalDecision = "deny"                │
   │       → SettlementAndClosure                             │
   │     "escalate":                                          │
   │       set case.triggerStage    = "eligibility"           │
   │       set case.reviewRequired  = true                    │
   │       → ClaimReview                                      │
   └──────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │ STAGE: AwaitingAssessment (secondary)                    │
   │                                                          │
   │   set case.currentStage = "AwaitingAssessment"           │
   │   set case.status       = "awaiting_assessment"          │
   │                                                          │
   │   Assessor Intake Robot job(case)                        │
   │                                                          │
   │   wait until case.assessmentReportPdf != null            │
   │     (poll interval: 30 minutes; timer: 10 business days; │
   │      on timer fire, set case.reviewRequired = true and   │
   │      continue waiting — no auto-escalation in v1)        │
   │                                                          │
   │   → DataAnalysis                                         │
   └──────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │ STAGE: DataAnalysis                                      │
   │                                                          │
   │   set case.currentStage = "DataAnalysis"                 │
   │   set case.status       = "in_progress"                  │
   │                                                          │
   │   SUB-STEP 1 — Validate                                  │
   │     invoke 02_assessment_validation with                 │
   │       in_ClaimDataJSON            = case.out_ClaimDataJSON
   │       in_PolicyDataJSON           = case.out_PolicyDataJSON
   │       in_AssessorReportPDF        = case.assessmentReportPdf
   │       in_ClaimEligibilityJSON     = case.out_ClaimEligibilityJSON
   │       in_EscalationDecision_Eligibility = case.out_EscalationDecision_Eligibility ?: null
   │       in_EscalationComment_Eligibility  = case.out_EscalationComment_Eligibility  ?: null
   │                                                          │
   │     set case.out_AssessmentValidationJSON = agent.02_*.out_AssessmentValidationJSON
   │     set case.out_AssessmentReportJSON     = agent.02_*.out_AssessmentReportJSON
   │                                                          │
   │     if agent.02_*.out_AssessmentValidationJSON.recommendation == "reject_report":
   │       set case.triggerStage   = "decision"               │
   │       set case.reviewRequired = true                     │
   │       → ClaimReview                                      │
   │                                                          │
   │   SUB-STEP 2 — Parallel block                            │
   │                                                          │
   │     ┌── parallel ──────────────────────────────────────┐ │
   │     │                                                  │ │
   │     │  invoke 03_coverage_analysis                     │ │
   │     │  invoke 04_payout_calculation                    │ │
   │     │  invoke 05_credibility_assessment                │ │
   │     │                                                  │ │
   │     │  (all three receive the same input set:          │ │
   │     │     ClaimDataJSON, PolicyDataJSON,               │ │
   │     │     AssessmentReportJSON, ClaimEligibilityJSON,  │ │
   │     │     AssessmentValidationJSON,                    │ │
   │     │     EscalationDecision_Eligibility,              │ │
   │     │     EscalationComment_Eligibility;               │ │
   │     │     05_* additionally receives                   │ │
   │     │     PriorClaimsJSON = case.priorClaims)          │ │
   │     │                                                  │ │
   │     │  join when all three complete                    │ │
   │     │                                                  │ │
   │     └──────────────────────────────────────────────────┘ │
   │                                                          │
   │     set case.out_CoverageAnalysisJSON     = agent.03_*.out_CoverageAnalysisJSON
   │     set case.out_PayoutCalculationJSON    = agent.04_*.out_PayoutCalculationJSON
   │     set case.out_CredibilityAssessmentJSON = agent.05_*.out_CredibilityAssessmentJSON
   │                                                          │
   │   SUB-STEP 3 — Synthesise                                │
   │     invoke 06_decision with all five envelopes plus     │
   │       in_EscalationDecision_Eligibility, in_EscalationComment_Eligibility
   │                                                          │
   │     set case.out_DecisionJSON   = agent.06_*.out_DecisionJSON
   │     set case.out_FinalDecision  = agent.06_*.out_FinalDecision
   │                                                          │
   │     Notification Robot job(case.claimantEmail, "Analysis complete")
   │                                                          │
   │     branch on case.out_FinalDecision:                    │
   │       "approve" | "partial_approve" | "deny":            │
   │         → SettlementAndClosure                           │
   │       "escalate":                                        │
   │         set case.triggerStage    = "decision"            │
   │         set case.reviewRequired  = true                  │
   │         → ClaimReview                                    │
   └──────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │ STAGE: ClaimReview (conditional primary)                 │
   │                                                          │
   │   set case.currentStage = "ClaimReview"                  │
   │   set case.status       = "escalated"                    │
   │                                                          │
   │   create Action Center task with payload assembled per   │
   │   action-center-tasks.md (uses case.triggerStage)        │
   │                                                          │
   │   wait for task completion                               │
   │     (timer: 5 business days; on expiry, alert ops queue) │
   │                                                          │
   │   on completion:                                         │
   │     let stage = case.triggerStage                        │
   │     let dec   = task.decision                            │
   │     let cmt   = task.reviewerNotes                       │
   │                                                          │
   │     if stage == "eligibility":                           │
   │       set case.out_EscalationDecision_Eligibility    = dec
   │       set case.out_EscalationComment_Eligibility     = cmt
   │       set case.out_EscalationReviewedAt_Eligibility  = task.reviewedAt
   │     if stage == "decision":                              │
   │       set case.out_EscalationDecision_Decision       = dec
   │       set case.out_EscalationComment_Decision        = cmt
   │       set case.out_EscalationReviewedAt_Decision     = task.reviewedAt
   │                                                          │
   │     set case.triggerStage   = null                       │
   │     set case.reviewRequired = false                      │
   │                                                          │
   │   branch on (stage, dec):                                │
   │     ("eligibility", "continue"):                         │
   │       if case.assessmentReportPdf != null                │
   │         → DataAnalysis                                   │
   │       else                                               │
   │         → AwaitingAssessment                             │
   │     ("eligibility", "deny"):                             │
   │       set case.out_FinalDecision = "deny"                │
   │       → SettlementAndClosure                             │
   │     ("decision", "continue"):                            │
   │       (final decision is now case.out_DecisionJSON.details.effective_decision_if_continued)
   │       → SettlementAndClosure                             │
   │     ("decision", "deny"):                                │
   │       set case.out_FinalDecision = "deny"                │
   │       → SettlementAndClosure                             │
   └──────────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │ STAGE: SettlementAndClosure                              │
   │                                                          │
   │   set case.currentStage = "SettlementAndClosure"         │
   │                                                          │
   │   invoke 07_claim_response with the full input set       │
   │   (see stages.md → Settlement and Closure)               │
   │                                                          │
   │   set case.out_ClaimResponseJSON = agent.07_*.out_ClaimResponseJSON
   │                                                          │
   │   Settlement Robot job(case, out_ClaimResponseJSON,      │
   │                        out_PayoutCalculationJSON,        │
   │                        out_DecisionJSON)                 │
   │   → robot performs: send letter email, post Finance Queue
   │     message if applicable, set case.status and closedAt  │
   │                                                          │
   │   wait until case.status in {approved, partial_approved, │
   │                              denied}                     │
   │                                                          │
   │   set case.currentStage = "Closed"                       │
   │   → end of process                                       │
   └──────────────────────────────────────────────────────────┘
```

---

## 5. The parallel block — sync details

Maestro's parallel activity fans out three branches and waits for the join. Each branch invokes one agent. The join must wait for **all three** before transitioning to the synthesise step.

Failure semantics:

| Scenario | Behaviour |
|---|---|
| One branch fails permanently after retries | Synthesise a critical-failure envelope for that agent and join with the others. Agent 06 will see the synthetic envelope and almost certainly emit `escalate`. |
| All three branches fail | Set `case.reviewRequired = true`, write a process-level error, transition to `ClaimReview` with `triggerStage = "decision"`. |
| One branch is slow (> 60s) | Allowed up to the agent's soft deadline. The other branches do not block on it. |
| Branches have differing inputs | This is a contract violation — all three receive the same input set per stages.md. The Maestro implementation must use a single input-assembly block whose output is fed identically to all three branches. |

---

## 6. Wait states and timers

Maestro has three wait states and three timer types in this process:

| Stage | Wait condition | Timer | On timer fire |
|---|---|---|---|
| `AwaitingAssessment` | `case.assessmentReportPdf != null` | 10 business days | Set `case.reviewRequired = true`, send follow-up email, continue waiting. |
| `DetailsInquiry` | `out_DetailsInquiryJSON.status in {responded, timed_out}` | 14 calendar days deadline; 7 calendar days reminder | Reminder: send reminder email. Deadline: set status to `timed_out`, transition back to originating stage with `case.reviewRequired = true`. |
| `ClaimReview` | Action Center task completed | 5 business days | Alert the Claims Operations Manager queue. Continue waiting. |

Timer values are read from configuration at process start so the process can be reconfigured per environment without redeployment.

---

## 7. Routing scalars

Maestro branches on two scalars only:

| Scalar | Producer | Branch points |
|---|---|---|
| `out_isEligible` | Agent 01 | None directly — branching is on `out_ClaimEligibilityJSON.recommendation` which conveys richer information. The scalar exists for the apps and for audit. |
| `out_FinalDecision` | Agent 06 | The exit of `DataAnalysis`. |

Inside `ClaimReview`, branching is on the composite `(case.triggerStage, task.decision)`.

Every other branch uses `<envelope>.recommendation`. The reason: scalars are quick to read in Maestro Studio, but `recommendation` carries the agent's full enum, which is more expressive when generating the process visually.

---

## 8. Agent invocation defaults

Apply to every agent invocation in the process unless the stage spec says otherwise:

| Concern | Default |
|---|---|
| Soft deadline | 60 seconds (120 seconds for Agent 07) |
| Retries on transient failure | 2 retries with exponential backoff (1s, 4s) |
| Model | Per `CONFIG.md` §1 (default_model or reasoning_model, depending on agent) |
| Trust Layer | Enabled |
| Token / cost cap | None at the process level — managed at the tenant level |
| Inputs assembly | Maestro Sequence activity that reads from case entity and constructs the input object |
| Output handling | Maestro Sequence activity that writes outputs to case entity in a single Update Row call |

---

## 9. Idempotency

Every Update Row activity must be idempotent against a single case key (`claimId`). The intake robot uses an upsert pattern: if a case row with the same `claimId` already exists, it overwrites the row rather than failing. This protects against duplicate intake emails or replayed Orchestrator queue messages.

Agents are stateless and idempotent: invoking the same agent twice with the same inputs produces an equivalent output. The Maestro process tolerates retries.

The Settlement Robot is idempotent on a per-case basis: it checks `case.letterEmailMessageId` and `case.paymentInstructionRef` before acting. If the message has already been sent or the instruction already posted, the robot logs and skips.

---

## 10. Auditability

Every Maestro activity emits a structured event to the AI Trust Layer and to the Orchestrator job log. The Process App's Timeline view (future) reads from this log.

Key event types:

| Event | When emitted | Carries |
|---|---|---|
| `stage_entered` | On every stage transition | from_stage, to_stage, timestamp |
| `agent_invocation` | Before each agent call | agent_slug, model, inputs_digest |
| `agent_completion` | After each agent call | agent_slug, status, recommendation, token_usage |
| `task_created` | When an Action Center task is created | task_id, trigger_stage, payload_digest |
| `task_completed` | When a task is completed | task_id, decision, reviewer_notes_digest |
| `email_sent` | After each Integration Service send | message_id, recipient, template |
| `queue_message_posted` | After Finance Queue write | queue_name, message_id |
| `case_closed` | On final transition | final_status, closedAt |

---

*End of maestro-process.md.*
