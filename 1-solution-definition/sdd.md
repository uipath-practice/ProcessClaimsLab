# Property Insurance Claims — Solution Definition Document (SDD)

|                                               |                                                                              |
| --------------------------------------------- | ---------------------------------------------------------------------------- |
| **Owner**                                     | Solutions Engineering, Apex Mutual Insurance Company                         |
| **Version**                                   | 1.0                                                                          |
| **Effective date**                            | 26 May 2026                                                                  |
| **Review cycle**                              | 12 months                                                                    |
| **Companion document**                        | `0-process-definition/pdd.md` (the business process this solution automates) |
| **Downstream specifications** (in subfolders) | `2-data-format/`<br>`2-case-management/`<br>`3-agents/`<br>`4-apps/`         |

---

## 1. Purpose

This document defines the **target solution architecture** for the Property Insurance Claims process specified in the PDD. It explains how each business stage, validation rule, and decision is realised on the **UiPath Agentic Automation Platform**.

The SDD is the bridge between the business process definition document (PDD) and the implementation artefacts (agent prompts, app code, case-management spec, data schemas). It does not prescribe code; it prescribes the components, contracts, and integrations that the downstream specifications must conform to.

---

## 2. Solution objectives

| #   | Objective                                                 | How the solution meets it                                                                                                                                 |
| --- | --------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Automate the adjudication of clean claims end to end      | An agent pipeline runs the Eligibility, Coverage, Payout, Credibility, Decision, and Communication steps without manual intervention.                     |
| 2   | Escalate ambiguous claims to a senior reviewer            | When an agent recommends escalation, Maestro Case Management creates an Action Center task that surfaces the full analysis in a dedicated review app.     |
| 3   | Preserve a complete audit trail                           | Every agent output is persisted on the case record; every check is identifiable by a stable rule ID; reviewer notes are stored alongside agent reasoning. |
| 4   | Render every agent's analysis through a single generic UI | Every agent emits the same unified output envelope (§6.3) so the apps render any agent's result through one shared component.                             |
| 5   | Treat documents, currencies, and locales as data          | The solution makes no assumptions about insurer, currency, locale, or date format; everything is derived from the documents and the policy in force.      |
| 6   | Keep humans in control                                    | A Senior Claims Adjuster may continue or deny any escalated claim with full discretion; the system never refuses a human decision.                        |

---

## 3. UiPath platform component map

| Capability                                              | Role in this solution                                                                                                                                                                                                            |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Document Understanding (IXP)**                        | Classifies the three claim documents; extracts structured data from the Claim Form, Insurance Policy, and Professional Incident Report.                                                                                          |
| **AI Agents (Agent Builder)**                           | Hosts agents that perform eligibility, validation, coverage, payout, credibility, decision, and communication.                                                                                                                   |
| **Maestro Case Management (Process Orchestrator)**      | Drives the case lifecycle. Owns stage transitions, the parallel block in Data Analysis, the escalation gates, and the wait points for the secondary stages.                                                                      |
| **Case Management (Data Service / Data Fabric entity)** | Backs each claim with a persistent record (`PropertyClaimCase`). Holds header fields, lifecycle state, agent outputs, attachments, and prior-claims data.                                                                        |
| **Action Center**                                       | Surfaces escalated claims to the Senior Claims Adjuster as tasks bound to the **Claim Review** Coded Action App.                                                                                                                 |
| **Coded Apps**                                          | Two custom applications: a **Process App** (operations dashboard) and an **Action App** (single-claim review form). Both consume a shared component library so that any agent's output renders identically across both surfaces. |
| **Storage Buckets**                                     | Store the three PDFs per claim (policy, claim form, assessor report).                                                                                                                                                            |
| **Integration Service**                                 | Delivers email notifications to the claimant via a managed email connector.                                                                                                                                                      |
| **Unattended Robots (RPA)**                             | Intake automation (document arrival, policy lookup, prior-claims retrieval), email dispatch, payment-instruction posting, and case archival.                                                                                     |
| **AI Trust Layer**                                      | Governs agent prompt/response handling, PII redaction policies, and model usage telemetry.                                                                                                                                       |
| **Orchestrator**                                        | Hosts robots, queues, and storage bucket access; surfaces job attachments.                                                                                                                                                       |

---

## 4. Solution architecture

```
   Claimant                       Inspector                  Claims Operations
   (Email)                        (Report upload)             (Process App)
       │                               │                           │
       ▼                               ▼                           │
   ┌────────────────────────────────────────────────────┐          │
   │            Intake Robot (Unattended)               │          │
   │  • Receives claim package (FNOL + policy ref +     │          │
   │    incident report when available)                 │          │
   │  • Uploads PDFs to Storage Buckets                 │          │ 
   │  • Looks up policy and prior claims                │          │
   │  • Creates PropertyClaimCase                       │          │
   └────────────────────┬───────────────────────────────┘          │
                        ▼                                          │
   ┌─────────────────────────────────────────────┐                 │
   │    Document Understanding (IXP)             │                 │
   │  • Classify document types                  │                 │
   │  • Extract structured fields                │                 │
   └─────────────────────┬───────────────────────┘                 │
                         ▼                                         │
   ┌─────────────────────────────────────────────────────────────┐ │
   │              Maestro Process Orchestrator                   │ │
   │                                                             │ │
   │   ┌────────────┐                                            │ │
   │   │ Agent 01   │ Claim Eligibility ── 5 threshold checks    │ │
   │   └─────┬──────┘                                            │ │
   │         │  (escalate) ─► Action Center (Claim Review,       │ │
   │         │                  eligibility trigger)             │ │
   │         ▼                                                   │ │
   │   ┌────────────┐                                            │ │
   │   │ Agent 02   │ Assessment Validation                      │ │
   │   └─────┬──────┘                                            │ │
   │         ▼                                                   │ │
   │   ┌────────────┬────────────┬────────────┐                  │ │
   │   │ Agent 03   │ Agent 04   │ Agent 05   │  parallel block  │ │
   │   │ Coverage   │ Payout     │ Credibility│                  │ │
   │   └─────┬──────┴─────┬──────┴─────┬──────┘                  │ │
   │         └────────────┼────────────┘                         │ │
   │                      ▼                                      │ │
   │   ┌────────────┐                                            │ │
   │   │ Agent 06   │ Decision                                   │ │
   │   └─────┬──────┘                                            │ │
   │         │  (escalate) ─► Action Center (Claim Review,       │ │
   │         │                  decision trigger)                │ │
   │         ▼                                                   │ │
   │   ┌────────────┐                                            │ │
   │   │ Agent 07   │ Claim Response — letter                    │ │
   │   └────────────┘                                            │ │
   └──────────────────────┬──────────────────────────────────────┘ │
                          ▼                                        │
   ┌──────────────────────────────────────┐                        │
   │  Settlement Robot (Unattended)       │                        │
   │  • Sends letter email (Integration   │                        │
   │    Service)                          │                        │
   │  • Posts payment instruction to      │                        │
   │    finance queue                     │                        │
   │  • Updates case status to closed     │                        │
   └──────────────────────────────────────┘                        │
                                                                   │
   ┌──────────────────────────────────────────────────────────┐    │
   │           Case Management Entity (Data Service)          │ ◄──┘
   │           PropertyClaimCase — single record per claim    │
   │           Header fields, agent outputs, attachments      │
   └──────────────────────────────────────────────────────────┘
                          ▲
                          │ (writes from every component)
                          
   Cross-cutting:
   • Storage Buckets ─── policy / claim form / assessor report PDFs
   • AI Trust Layer ─── governs agent prompt/response, PII
   • OAuth (External App) ─── authenticates the two coded apps
```

---

## 5. Process-to-component mapping

This section maps every PDD stage onto the UiPath components that realise it. Numbering follows the PDD.

### 5.1 Stage 1 — Intake

| Aspect                | Mapping                                                                                                                                                                                                                                                                                                                                                           |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Trigger**           | Claim packet arrives via email inbox monitored by an unattended robot, or by direct upload through Apex Mutual's claimant portal.                                                                                                                                                                                                                                 |
| **Components**        | Unattended Robot (intake) → IXP & Storage Buckets                                                                                                                                                                                                                                                                                                                 |
| **Sequence**          | (1) Robot picks up the package; (2) Storage Buckets receive the PDFs; (3) IXP classifies and extracts documents; (4) Robot creates the `PropertyClaimCase` record with extracted header data; (5) Robot looks up the policy by ID and the claimant's prior claims and writes them to the case; (6) Robot sends the acknowledgement email via Integration Service. |
| **Maestro role**      | Maestro starts the case journey at stage `Intake` and transitions it to `Eligibility Analysis` on completion.                                                                                                                                                                                                                                                     |
| **Outputs persisted** | Header fields; `claimFormPdf`, `policyPdf`, `assessmentReportPdf` (when present); `out_ClaimIXPDataJSON`; `priorClaims[]`.                                                                                                                                                                                                                                        |

### 5.2 Stage 2 — Eligibility Analysis

| Aspect                | Mapping                                                                                                                                                                                                                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Component**         | 01 Claim Eligibility Agent                                                                                                                                                                                                                                                                        |
| **Inputs**            | `in_PolicyID`, `in_ClaimIXPDataJSON`, `in_PolicyPDF`                                                                                                                                                                                                                                              |
| **Outputs persisted** | `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_ClaimEligibilityJSON`, `out_ClaimEligibilitySummary`                                                                                                                                                                                              |
| **Escalation path**   | When the agent returns `recommendation = escalate`, Maestro creates an Action Center task targeting the **Claim Review** Coded Action App with `triggerStage = "eligibility"`. The reviewer's outcome (`continue` or `deny`) is written back to the case as `out_EscalationDecision_Eligibility`. |
| **Auto-deny path**    | When the agent returns `recommendation = deny`, Maestro transitions the case directly to `Settlement & Closure`.                                                                                                                                                                                  |
| **Email**             | The robot sends a status email after the 01 Claim Eligibility Agent completes if the claim is found eligible for analysis.                                                                                                                                                                        |

### 5.3 Awaiting Assessment (Secondary stage)

| Aspect        | Mapping                                                                                                                                                                                                                                                                             |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Component** | Unattended Robot (assessment intake) + Storage Buckets                                                                                                                                                                                                                              |
| **Trigger**   | Eligibility passed and `assessmentReportPdf` is not present on the case.                                                                                                                                                                                                            |
| **Behaviour** | The robot issues the inspection request email to the assessor panel, monitors the assessor inbox / portal upload for the report PDF, uploads it to the `Assessor Reports` bucket, and updates the case attachment field. Maestro then transitions the case back to `Data Analysis`. |
| **Timer**     | Maestro maintains a configurable timer (default ten business days) after which the case is flagged for follow-up.                                                                                                                                                                   |

### 5.4 Details Inquiry (Secondary stage)

| Aspect | Mapping |
|---|---|
| **Component** | Unattended Robot (inquiry dispatch) + Maestro wait state |
| **Trigger** | Invoked from Intake (missing required document), Eligibility Analysis (clarification required), or Claim Review (additional information requested by the adjuster). |
| **Behaviour** | The robot sends an email to the claimant via Integration Service describing the information required and the response deadline. The case enters a Maestro wait state with a 14-day timer and a 7-day reminder. On receipt of a reply (or timer expiry) the case returns to the originating stage with the supplementary information attached. |
| **Persistence** | `out_DetailsInquiryJSON` records the inquiry context: originating stage, requested information, sent timestamp, response timestamp, and the claimant's response or a `timed_out` flag. |
| **Initial release note** | The end-to-end flow in the initial release does not automatically generate inquiries; the stage exists in the lifecycle and is reachable manually from the Action App's reviewer notes. Automatic inquiry generation is scheduled for a subsequent release. |

### 5.5 Stage 3 — Data Analysis

| Aspect                      | Mapping                                                                                                                                                                                                                                  |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Entry condition**         | Eligibility confirmed and `assessmentReportPdf` present.                                                                                                                                                                                 |
| **Sub-step 1 — Validate**   | The 02 Assessment Validation Agent runs against the assessor PDF, the policy data, and the eligibility result. Persists `out_AssessmentReportJSON`, `out_AssessmentValidationJSON`, `out_AssessmentValidationSummary`.                   |
| **Sub-step 2 — Parallel**   | Maestro spawns Agents 03, 04, and 05 in parallel. Each receives the same inputs (claim data, policy data, validated assessment, eligibility result, prior claims, reviewer notes if any). Each emits its own check set and recommendation. |
| **Sub-step 3 — Synthesise** | The 06 Decision Agent consumes all four upstream outputs and produces `out_FinalDecision` + `out_DecisionJSON`.                                                                                                                          |
| **Escalation path**         | If `out_FinalDecision = escalate`, Maestro creates an Action Center task targeting **Claim Review** with `triggerStage = "decision"`. Otherwise the case transitions directly to `Settlement & Closure`.                                 |
| **Email**                   | The robot sends a status email after the 06 Decision Agent completes.                                                                                                                                                                    |

### 5.6 Stage 4 — Claim Review

| Aspect                | Mapping                                                                                                                                                 |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Component**         | Action Center task bound to the **Claim Review** Coded Action App.                                                                                      |
| **Input payload**     | Full agent output set (eligibility, assessment, coverage, payout, credibility, decision) + the three PDFs as job attachments + the `triggerStage` flag. |
| **Reviewer outcomes** | `continue` or `deny`, plus optional `reviewerNotes` and `reviewedAt`.                                                                                   |
| **Authority**         | Fully open. The reviewer may continue a claim the agents wanted to deny and may deny a claim the agents wanted to approve.                              |
| **Persistence**       | `out_EscalationDecision_Decision` and `out_EscalationComment_Decision` are written to the case.                                                         |

### 5.7 Stage 5 — Settlement and Closure

| Aspect          | Mapping                                                                                                                                                                                                                                                                                                                                                                                       |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Component**   | 07 Claim Response Agent → Unattended Robot (settlement) → Integration Service → Orchestrator queue                                                                                                                                                                                                                                                                                            |
| **Sequence**    | (1) The 07 Claim Response Agent drafts the decision letter incorporating any reviewer comment; (2) the robot sends the letter by email via Integration Service to the address on file; (3) for approvals and partial approvals, the robot posts a payment instruction to the **Finance Payout Queue** in Orchestrator; (4) the robot updates the case status to `closed` and archives the record. |
| **Persistence** | `out_ClaimResponseJSON` (letter text + metadata) + `out_ClaimResponseSummary`; case `status = approved \| partial_approved \| denied \| closed`.                                                                                                                                                                                                                                              |

---

## 6. Agent pipeline

### 6.1 Agent roster

Naming convention: **`<ID> <Purpose> Agent`**. The slug column is used in file names (`NN_<slug>_agent_prompts.md`) and in PascalCase form for variables (`out_<AgentName>JSON`).

| #      | Agent                            | Slug                     | Stage           | Inputs (high-level)                                       | Primary outputs                                                                  | Recommendation values                                                              |
| ------ | -------------------------------- | ------------------------ | --------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **01** | Claim Eligibility Agent          | `claim_eligibility`      | 2               | Policy ID, Claim IXP data, Policy PDF                     | `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_ClaimEligibilityJSON`            | `proceed_to_coverage_analysis` \| `deny` \| `escalate`                              |
| **02** | Assessment Validation Agent      | `assessment_validation`  | 3 entry         | Claim data, Policy data, Assessor PDF, Eligibility result | `out_AssessmentReportJSON`, `out_AssessmentValidationJSON`                       | `proceed_to_parallel_analysis` \| `escalate` \| `reject_report`                     |
| **03** | Coverage Analysis Agent          | `coverage_analysis`      | 3 (parallel)    | Claim, Policy, Assessment, Eligibility, reviewer notes    | `out_CoverageAnalysisJSON`                                                        | `proceed_to_payout` \| `deny_all` \| `partial_approval` \| `escalate`               |
| **04** | Payout Calculation Agent         | `payout_calculation`     | 3 (parallel)    | Claim, Policy, Assessment, Eligibility, reviewer notes    | `out_PayoutCalculationJSON`                                                       | `approve_payout` \| `flag_for_review` \| `escalate`                                 |
| **05** | Credibility Assessment Agent     | `credibility_assessment` | 3 (parallel)    | Claim, Policy, Assessment, Eligibility, Prior claims, reviewer notes | `out_CredibilityAssessmentJSON`                                       | `proceed_to_decision` \| `escalate_to_human`                                        |
| **06** | Decision Agent                   | `decision`               | 3 (synthesis)   | All upstream agent outputs, reviewer notes                | `out_FinalDecision`, `out_DecisionJSON`                                          | `approve` \| `partial_approve` \| `deny` \| `escalate`                              |
| **07** | Claim Response Agent             | `claim_response`         | 5               | All upstream + optional reviewer outcome                  | `out_ClaimResponseJSON`                                                          | (drafts a letter; no recommendation)                                                |

Every agent additionally emits the plain-text `out_<AgentName>Summary` field carrying the same content as the envelope's `summary`.

Detailed prompts, payload schemas, and worked examples live in `3-agents/`. Model assignment per agent is in `CONFIG.md` §1.

### 6.2 Data flow

```
Intake (RPA + IXP for Claim Form only)
   │  in_PolicyID, in_ClaimIXPDataJSON, in_PolicyPDF
   ▼
[01] Claim Eligibility Agent
                        ──► out_ClaimDataJSON,
                            out_PolicyDataJSON,
                            out_ClaimEligibilityJSON,
                            out_ClaimEligibilitySummary
   │
   │  (proceed_to_coverage_analysis)
   ▼
[02] Assessment Validation Agent
                        ──► out_AssessmentReportJSON,
                            out_AssessmentValidationJSON,
                            out_AssessmentValidationSummary
   │
   ▼  (Maestro fork)
┌─────────────────────────── parallel block ───────────────────────────┐
[03] Coverage Analysis Agent      ──► out_CoverageAnalysisJSON,
                                      out_CoverageAnalysisSummary
[04] Payout Calculation Agent     ──► out_PayoutCalculationJSON,
                                      out_PayoutCalculationSummary
[05] Credibility Assessment Agent ──► out_CredibilityAssessmentJSON,
                                      out_CredibilityAssessmentSummary
└──────────────────────────────────────────────────────────────────────┘
   │  (Maestro join)
   ▼
[06] Decision Agent     ──► out_FinalDecision,
                            out_DecisionJSON,
                            out_DecisionSummary
   │
   │  (auto path: approve / partial / deny)
   │  (escalate path: Action Center → reviewer outcome → reviewerNotes
   │                  fed back into the case and propagated as input
   │                  to all downstream agents)
   ▼
[07] Claim Response Agent ──► out_ClaimResponseJSON (letter),
                              out_ClaimResponseSummary
   │
   ▼
Settlement (RPA)
```

### 6.3 Unified output envelope

Every agent emits the same outer JSON shape. This is the central design rule of the solution: it lets the apps render any agent's analysis through a single shared component without per-agent layouts.

The envelope at a glance:

```jsonc
{
  "out_<AgentName>JSON": {
    "claim_id":       "<string>",
    "status":         "ok | warn | critical",
    "recommendation": "<agent-specific recommendation enum>",
    "checks": [
      {
        "id":       "<stable rule ID, e.g. E-1, C-3, P-7, CR-2, D-1>",
        "label":    "<short human label>",
        "result":   "pass | fail | n/a",
        "severity": "info | warn | critical",
        "detail":   "<one-sentence outcome>",
        "evidence": [
          { "source": "<dot-path into source document, e.g. policy.declarations.payment_status>",
            "value":  "<observed value>" }
        ]
      }
    ],
    "findings":  [ "<short bullet, suitable for human display>" ],
    "summary":   "<plain-text reasoning paragraph; mirrored on out_<AgentName>Summary>",
    "details":   { /* agent-specific structured payload (coverage_determination,
                     section_totals, credibility_assessment, etc.) */ }
  },
  "out_<AgentName>Summary": "<plain-text duplicate of summary, surfaced directly by panels and emails>"
}
```

Notes:

- The shared envelope keys (`status`, `recommendation`, `checks`, `findings`, `summary`, `details`) are present on every agent output.
- `AgentName` is PascalCase derived from the agent slug (`claim_eligibility` → `ClaimEligibility`).
- The inner `details` block is the only agent-specific area. Apps render it through an "Inspect JSON" expander; structured renderers for `details` are added per agent only when there is a strong UX case.
- Agents that also emit auxiliary outputs (e.g. the 01 Claim Eligibility Agent emits `out_ClaimDataJSON` + `out_PolicyDataJSON`; the 07 Claim Response Agent emits `out_ClaimResponseJSON`) place those at the same level as the envelope, not inside it.
- When a reviewer override is present in the case, agents in stages after the override receive `reviewerNotes` as an input and must not re-raise concerns the reviewer has resolved (see `CONFIG.md` §10).
- The full schema, including the discriminated-union variants of `details` per agent, lives in `2-data-format/`.

### 6.4 Model selection and the AI Trust Layer

| Concern               | Approach                                                                                                                                                                                                                                                                                                          |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Model selection**   | `CONFIG.md` §1 defines two model roles (`default_model`, `reasoning_model`) and assigns one to each agent. Agent prompts reference the roles symbolically; concrete model IDs are not pinned in prompts. Coverage Analysis, Credibility Assessment, and Decision use the reasoning model; the rest use the default. |
| **PII handling**      | The AI Trust Layer is enabled for all agents. Claimant PII (name, address, email, phone) is allowed through because it is required for adjudication; financial account details and identifiers are out of scope.                                                                                                  |
| **Prompt versioning** | Agent prompts in `3-agents/` are the source of truth. Maestro pins the agent version at deployment time.                                                                                                                                                                                                          |
| **Telemetry**         | Token usage, latency, and model identity are written to AI Trust Layer logs and surfaced in the Process App via a future Timeline view.                                                                                                                                                                           |

### 6.5 Where the detail lives

`3-agents/` holds, per agent, two files keyed by the agent slug:

- `NN_<agent_slug>_agent_prompts.md` — Role, Inputs Provided, Instructions, Output Format, Special Rules, User Prompt.
- `NN_<agent_slug>_agent_payloads.md` — Inputs JSON schema, Output Format narrative, Output JSON schema, Sample Payload.

`NN` is the two-digit agent number (`01`..`07`). Example: `3-agents/03_coverage_analysis_agent_prompts.md`.

File naming conventions, the required `out_<AgentName>Summary` field, and the consistency checklist follow the rules in `inputs/agents input/agents.md` (carried forward into `3-agents/` with the new envelope contract applied).

---

## 7. Case management

### 7.1 Entity

The Data Service / Data Fabric entity is named `PropertyClaimCase`. Each claim is exactly one record. The entity is enriched stage by stage; every agent output is optional at the schema level because a case starts at Intake with only header data.

### 7.2 Lifecycle stages

| Stage code | Stage name | Type |
|---|---|---|
| `Intake` | Stage 1 — Intake | Primary |
| `EligibilityAnalysis` | Stage 2 — Eligibility Analysis | Primary |
| `AwaitingAssessment` | Awaiting Assessment | Secondary |
| `DetailsInquiry` | Details Inquiry | Secondary |
| `DataAnalysis` | Stage 3 — Data Analysis | Primary |
| `ClaimReview` | Stage 4 — Claim Review | Primary (conditional) |
| `SettlementAndClosure` | Stage 5 — Settlement and Closure | Primary |

### 7.3 Status enum

```
pending_review | in_progress | awaiting_assessment | awaiting_details |
escalated | approved | partial_approved | denied | closed
```

### 7.4 Key fields (high-level)

| Group                     | Fields                                                                                                                                         |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **Header (denormalised)** | `claimId` (key), `policyId`, `claimantName`, `currency`, `incidentType`, `incidentDate`, `dateOfSubmission`, `totalClaimAmount`                |
| **Lifecycle**             | `currentStage`, `status`, `updatedAt`, `reviewRequired`, `triggerStage`                                                                        |
| **Attachments**           | `claimFormPdf`, `policyPdf`, `assessmentReportPdf` (each as an attachment reference resolved against Storage Buckets)                          |
| **Reference data**        | `priorClaims[]` (denormalised summary of the claimant's prior claims)                                                                          |
| **Intake outputs**        | `out_ClaimIXPDataJSON`                                                                                                                            |
| **Agent 01**              | `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_ClaimEligibilityJSON`, `out_ClaimEligibilitySummary`, `out_isEligible`, `out_isComplete`           |
| **Agent 02**              | `out_AssessmentReportJSON`, `out_AssessmentValidationJSON`, `out_AssessmentValidationSummary`                                                     |
| **Agent 03**              | `out_CoverageAnalysisJSON`, `out_CoverageAnalysisSummary`                                                                                         |
| **Agent 04**              | `out_PayoutCalculationJSON`, `out_PayoutCalculationSummary`                                                                                       |
| **Agent 05**              | `out_CredibilityAssessmentJSON`, `out_CredibilityAssessmentSummary`                                                                               |
| **Agent 06**              | `out_FinalDecision`, `out_DecisionJSON`, `out_DecisionSummary`                                                                                    |
| **Agent 07**              | `out_ClaimResponseJSON`, `out_ClaimResponseSummary`                                                                                               |
| **Human review**          | `out_EscalationDecision_Eligibility`, `out_EscalationComment_Eligibility`, `out_EscalationDecision_Decision`, `out_EscalationComment_Decision`    |
| **Inquiry**               | `out_DetailsInquiryJSON`                                                                                                                          |

The detailed field list, types, and Data Service column mapping live in `2-case-management/`.

### 7.5 Lifecycle transitions

Transitions are driven by Maestro and recorded as case events for audit:

| From | To | Trigger |
|---|---|---|
| `Intake` | `EligibilityAnalysis` | All three documents (or at least Claim Form and Policy) classified and extracted |
| `Intake` | `DetailsInquiry` | Required document missing |
| `EligibilityAnalysis` | `AwaitingAssessment` | Eligibility pass and assessor report not yet on file |
| `EligibilityAnalysis` | `DataAnalysis` | Eligibility pass and assessor report present |
| `EligibilityAnalysis` | `SettlementAndClosure` | Eligibility hard deny |
| `EligibilityAnalysis` | `DetailsInquiry` | Reviewer requests clarification |
| `AwaitingAssessment` | `DataAnalysis` | Assessor report received |
| `DataAnalysis` | `SettlementAndClosure` | Auto decision (approve / partial / deny) |
| `DataAnalysis` | `ClaimReview` | Decision is `escalate` |
| `ClaimReview` | `SettlementAndClosure` | Reviewer outcome recorded |
| `ClaimReview` | `DetailsInquiry` | Reviewer requests additional information |
| `DetailsInquiry` | (originating stage) | Claimant response received or timer expires |
| `SettlementAndClosure` | (closed) | Letter sent, payment instruction issued, archive complete |

---

## 8. Coded applications

Two Coded Apps are built for this solution. Both consume a shared component library so that the same panel renders an agent's analysis identically in either surface.

### 8.1 Process App — `property-claims`

| Aspect | Value |
|---|---|
| **App type** | Coded **Web App** (CDN-served React app embedded in the UiPath portal) |
| **Audience** | Claims Operations team |
| **Routing name** | `property-claims` |
| **Routes** | `#/` and `#/claims` (Dashboard), `#/claims/<claimId>` (Claim Details) |
| **Read scope** | `PropertyClaimCase` (list + detail), Storage Buckets (PDF previews) |
| **Write scope** | None — the Process App is read-only |
| **Key views** | KPI cards (total, pending review, approved, denied); filterable claims table; claim detail page with the **TabbedAnalysis** shared component |

### 8.2 Action App — `claim-review`

| Aspect | Value |
|---|---|
| **App type** | Coded **Action App** (rendered inside an Action Center task) |
| **Audience** | Senior Claims Adjuster (single-claim review) |
| **Routing name** | `claim-review` |
| **Trigger modes** | `triggerStage = "eligibility"` (escalation from Stage 2) and `triggerStage = "decision"` (escalation from Stage 3) |
| **Input contract** | Defined in `4-apps/action-app/action-schema.json` — includes all agent outputs available at trigger time plus the three PDFs |
| **Output contract** | `decision: "continue" | "deny"`, optional `reviewerNotes`, `reviewedAt` (ISO 8601) |
| **Authority** | Fully open at both trigger stages. The reviewer may continue or deny any claim. |

### 8.3 Shared component library

A shared library powers both apps. Key components:

| Component | Purpose |
|---|---|
| **TabbedAnalysis** | Top-level tab bar (Claim, Policy, Eligibility, Assessment, Coverage, Payout, Credibility, Summary). One tab per agent output. Status badge per tab driven by the envelope's `status` field. |
| **GenericAnalysisPanel** | Renders any agent output by reading the unified envelope (status banner, findings list, checks table, summary, Inspect JSON expander). |
| **AgentDetailsPanel** | Pluggable inner panel that takes the agent's `details` block and renders an agent-specific layout (Coverage table, Payout breakdown, etc.). Falls back to JSON tree when no panel is registered for an agent. |
| **SummaryBlocks** | Three-card summary above the tabs: Claim Summary, Policy Summary, Assessment Summary. Each card surfaces key facts and a "Preview PDF" button. |
| **DecisionActionBar** | Two-stage Continue / Deny confirmation in the Action App. Hidden in the Process App. |
| **LifecycleBar** | Visualises the case lifecycle and current stage. Highlights conditional stages when entered. |

The shared library, panel registry, and OAuth/auth bootstrap live in `4-apps/shared/`.

### 8.4 OAuth and scopes

Both apps authenticate via a single External App on the Apex Mutual UiPath tenant using OAuth 2.0 with PKCE. Scopes:

```
OR.Folders                  # Storage Bucket reads via Orchestrator
OR.Administration.Read      # Tenant metadata
DataFabric.Data.Read        # PropertyClaimCase reads
DataFabric.Schema.Read      # Entity discovery
```

The Action App also relies on the Action Center session token surfaced by the SDK; explicit scopes only matter when calling Storage Buckets for previews.

---

## 9. Document Understanding

### 9.1 Document types

| Type | Source | Extraction approach |
|---|---|---|
| **Claim Form (FNOL)** | Claimant submission | Structured-form extraction; fields published as `in_ClaimIXPDataJSON` |
| **Insurance Policy** | Apex Mutual records | Reasoned extraction (Generative Extractor) because the policy contains sectioned legal text that cannot be reliably extracted by template alone |
| **Professional Incident Report** | Independent assessor | Reasoned extraction; the report is semi-structured with narrative sections |

### 9.2 Extraction strategy

The initial release uses the **Generative Extractor** capability in IXP. This avoids the cost of training a custom ML extraction model and tolerates variation in claim form layouts, policy schedules, and assessor report templates. A confidence threshold of **80 %** applies to each extracted field; fields below the threshold are surfaced for Validation Station review before the case advances.

The full field catalogue per document type is in `2-data-format/document-fields.md`.

> **Note for coding agents.** The implementation differs from the conceptual description above in one important way: **only the Claim Form is sent to IXP**. The Insurance Policy PDF is passed directly to the **01 Claim Eligibility Agent** as an input PDF, and the Professional Incident Report PDF is passed directly to the **02 Assessment Validation Agent**. Each of those agents extracts the fields it needs into its own JSON output (`out_PolicyDataJSON`, `out_AssessmentReportJSON`), with `evidence[]` entries citing the source document. The descriptions in §9.1 / §9.2 should be read as *"the data must end up in JSON"* — in code, the agent itself produces that JSON rather than IXP. See `CONFIG.md` §8 for the routing table.

### 9.3 Data normalisation

After extraction, the following deterministic normalisations are applied before any rule evaluates the data:

| Field type | Normalisation |
|---|---|
| **Date** | Parse to ISO 8601 (`YYYY-MM-DD`). Source documents may use `DD/MM/YYYY` or `MM/DD/YYYY`; format is inferred from the document's locale. |
| **Currency amount** | Parse to a `{ amount: number, currency: ISO-4217 code }` pair. Symbols are mapped to currency codes (`$` → policy currency context, `HK$` → `HKD`, `€` → `EUR`, etc.). |
| **Address** | Lowercase, collapse whitespace, expand common abbreviations (`St` → `Street`, `Apt` → `Unit`), strip punctuation. Used only for rule **E-3** comparison; the original strings remain in the case for display. |
| **Boolean strings** | Map `"Yes"/"No"`, `"True"/"False"`, `"1"/"0"` to booleans. |

Normalisation rules live in `2-data-format/normalisation.md`.

---

## 10. Integration glue

The unattended robots handle the integrations that sit between the agentic pipeline and the outside world.

### 10.1 Email notifications

All claimant communications are delivered by email through the **UiPath Integration Service Gmail connector** (see `CONFIG.md` §4). The robot resolves the claimant's address from `out_ClaimDataJSON.ClaimClaimant.EmailAddress`. The locale of the message body is derived from the claimant's address country (see PDD §11).

| Touchpoint | Email |
|---|---|
| Intake | Acknowledgement with claim reference number |
| Eligibility outcome | Status update (under review / proceeding to inspection / denied) |
| Awaiting Assessment | Inspection scheduled / report received |
| Data Analysis | Analysis complete; decision forthcoming |
| Details Inquiry | Request for additional information; reminder at day 7 |
| Settlement | Final decision letter |
| Closure | Case closed confirmation |

In pre-production environments email is delivered to a development inbox (one address per environment) so that the entire flow can be exercised end to end without contacting real claimants.

### 10.2 Payment instruction

Approvals and partial approvals trigger a message to the **Finance Payout Queue** in Orchestrator. Message payload:

```jsonc
{
  "claimId":          "string",
  "policyId":         "string",
  "claimantName":     "string",
  "claimantEmail":    "string",
  "currency":         "ISO-4217 code",
  "netPayoutAmount":  "number",
  "settlementBasis":  "replacement_cost | actual_cash_value",
  "approvedAt":       "ISO-8601 timestamp",
  "reference":        "ClaimResponseJSON.letter_metadata reference"
}
```

The Finance ERP integration is owned by Finance Engineering and out of scope for this solution. In pre-production environments a stub consumer drains the queue and logs the messages.

### 10.3 Archival

Apex Mutual's data retention policy for property claims is seven years. The case record in Data Service is the system of record; PDFs are retained in Storage Buckets. Archive lifecycle policies are configured on the buckets directly.

### 10.4 Document sourcing during pre-production

Initial test data is produced by a synthetic-document generator (templates under `inputs/document samples/Templates/property-claims/`). The generator produces batches of one Claim Form, one Insurance Policy, and one Professional Incident Report at a time, varying:

- Currency, locale, and claimant identity
- Incident type (wind, fire, water, theft, vandalism, ...)
- Pre-tagged adverse conditions (the discrepancy catalogue in PDD §10)

The agents make no use of the generator tags; tags are used to verify that agents produce the expected outcome.

---

## 11. Environments, identity, and deployment

### 11.1 Environments

| Environment | Tenant slug         | Purpose |
|---|---|---|
| Development | `apexmutual-dev`   | Iterative development; mock data only |
| Test        | `apexmutual-test`  | End-to-end testing against the synthetic-document generator and stub integrations |
| Production  | `apexmutual-prod`  | Live claims processing |

Each environment is a separate UiPath tenant under the Apex Mutual organisation. Environment-specific values (URLs, client IDs, bucket names) are kept in deployment configuration outside this document. The full configuration table is in `CONFIG.md` §2.

### 11.2 External App registration

A single External App is registered per environment for both Coded Apps. Authentication uses OAuth 2.0 with PKCE; no client secret. Redirect URIs include the deployed app URLs and `http://localhost:5173` for local development.

### 11.3 Storage Buckets

Three buckets are provisioned in Orchestrator:

| Bucket name | Contents |
|---|---|
| `Insurance Policies Repository` | Insurance Policy PDFs |
| `Claims` | First Notice of Loss PDFs |
| `Assessor Reports` | Professional Incident Report PDFs |

Bucket names are environment-configurable and surfaced through the apps' runtime configuration.

### 11.4 Deployment pipeline

The Coded Apps are packaged and published with the UiPath CLI:

```
uip codedapp pack dist -n <app-name> -v <version>
uip codedapp publish [-t Action]   # -t Action required for the Action App
uip codedapp deploy
```

Agent prompts and payloads are deployed via the UiPath Agents CLI from `3-agents/`. The Maestro process and Data Service entity are deployed via the standard Studio publish path.

---

## 12. Non-functional requirements

### 12.1 Performance and SLA

| Path | Target | Notes |
|---|---|---|
| Automated (no escalation, no human in loop) | 5–10 minutes intake to letter | Bound by agent latency and IXP extraction; no human gates |
| Single human escalation (Stage 2 or Stage 4) | 1–3 business days | Bound by reviewer SLA |
| Two human escalations + Details Inquiry | up to 14 calendar days | Bound by claimant response window |

Per-agent latency budget (target): under 60 seconds for Eligibility, Coverage, Payout, Credibility, Decision; up to 120 seconds for Claim Response (drafting).

### 12.2 Observability

- AI Trust Layer captures every agent invocation (prompt hash, model, token usage, latency, output digest).
- Orchestrator logs every robot action with structured payload references.
- Maestro records every stage transition with timestamps and triggering event.
- The Process App surfaces a Timeline view per claim summarising agent runs and stage transitions (in a subsequent release).

### 12.3 Audit

Every check carried out by an agent has a stable rule ID (E-1, C-3, P-7, CR-2, D-1, etc.) and a citation back to the source field (`evidence` array on each check). Compliance can reconstruct any decision by reading the case record alone.

### 12.4 Security and PII

- All data resides within the Apex Mutual UiPath tenant.
- The apps authenticate via OAuth PKCE; no static credentials.
- The AI Trust Layer governs which fields are permitted in prompts; financial account identifiers are blocked at the layer.
- Email transport uses the Integration Service's managed connector (TLS).
- Storage Buckets enforce read-only access for the apps; only the unattended robots have write access.

### 12.5 Resilience

- **Agent failure** — Maestro retries a failed agent invocation twice with exponential backoff. A persistent failure escalates the claim with a synthetic agent output that surfaces a critical blocking flag.
- **IXP confidence below threshold** — Validation Station gates the case at Intake; the case does not progress to Eligibility Analysis until the field is reviewed.
- **Email delivery failure** — Email failures are queued for retry; the case stage does not advance until the dispatch is confirmed.
- **Action Center task timeout** — Maestro applies a five-business-day timeout; on expiry the case is escalated to a Claims Operations Manager queue.

---

## 13. Architectural decision log

All architectural decisions below are **resolved**. They are recorded here as a reference for downstream specs and code generation.

| #       | Decision                                              | Resolution                                                                                                                                                                                                                                                                                                                                            |
| ------- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A1**  | IXP extractor type for the Claim Form                 | **Resolved: Generative Extractor.** No model training overhead; tolerates document variation. Applies only to the Claim Form — the Policy and Incident Report are read directly by agents (see §9.2 note and `CONFIG.md` §8).                                                                                                                          |
| **A2**  | Email connector through Integration Service           | **Resolved: Gmail connector** (UiPath Integration Service). Sender per environment; in Dev / Test, all mail is routed to a mock inbox. See `CONFIG.md` §4.                                                                                                                                                                                            |
| **A3**  | LLM model assignment per agent                        | **Resolved: model IDs are configuration, not prompt content.** `CONFIG.md` §1 defines two roles (`default_model`, `reasoning_model`) and a per-agent assignment. Prompts reference roles symbolically. Coverage Analysis (03), Credibility Assessment (05), and Decision (06) use the reasoning model; the rest use the default.                       |
| **A4**  | Date formats                                          | **Resolved: ISO 8601 on the case entity; `DD/MM/YYYY` in agent JSON inputs / outputs.** Agents read whatever format the source documents carry; the case-write layer normalises. The synthetic-document generator currently emits `DD/MM/YYYY` and is the lever for any future format change. See `CONFIG.md` §5.                                       |
| **A5**  | Currency on the case entity                           | **Resolved: denormalised header field `currency`** populated at Intake from the Policy data. Agents continue to source it from the policy at reasoning time. No FX conversion anywhere (see `CONFIG.md` §6).                                                                                                                                          |
| **A6**  | `out_DetailsInquiryJSON` shape                        | **Resolved: a single envelope-shaped object** that records originating stage, requested information, sent / response / timeout timestamps. Defined in `2-data-format/`.                                                                                                                                                                                |
| **A7**  | Process App write-back                                | **Resolved: read-only in v1.** A subsequent release may add a "Reopen case" action restricted to Claims Operations Managers.                                                                                                                                                                                                                          |
| **A8**  | Action App contract for human override                | **Resolved: 2-way (`continue` / `deny`) with free-text `reviewerNotes`** at both escalation points (eligibility and decision). The reviewer's `reviewerNotes` is **propagated as an input to all downstream agents**; agents treat the human decision as authoritative and must not re-raise concerns the reviewer has resolved (see `CONFIG.md` §10). |
| **A9**  | Storage Bucket access                                 | **Resolved.** UiPath built-in Storage Bucket activities serve agents and robots. Coded Apps use short-lived signed URLs via `GetReadUri` for the React PDF viewer (Apps cannot use the same server-side API). Buckets are never directly exposed.                                                                                                      |
| **A10** | Audit export                                          | **Resolved: no separate audit-export pipeline.** Every check, evidence record, summary, decision, and reviewer note is stored in Data Fabric on `PropertyClaimCase`. The Process App provides search and filter; auditors review individual records there. See `CONFIG.md` §13.                                                                        |

Locked: A1–A10 as of 2026-05-27.

---

## 14. Glossary (UiPath terms)

| Term | Meaning |
|---|---|
| **Maestro** | UiPath Process Orchestrator — drives multi-step automation workflows including agent invocations and case-management state transitions |
| **Agent Builder** | The UiPath surface for authoring AI Agents and deploying them as callable units within Maestro |
| **AI Trust Layer** | Governance layer over agent prompts and responses (PII redaction, model usage logging, content policy) |
| **Action Center** | UiPath surface for human-in-the-loop tasks; hosts the Action App rendered to a single reviewer |
| **Coded App** | A custom React application packaged and hosted by UiPath, either as a Web App (operations dashboard) or an Action App (Action Center form) |
| **Data Service / Data Fabric** | UiPath entity store — Data Service is the API; Data Fabric is the broader unified data plane |
| **Integration Service** | UiPath surface for managed third-party connectors (email, calendar, CRM, etc.) |
| **IXP (Intelligent Xtraction Platform)** | UiPath Document Understanding capability for classification and extraction |
| **Orchestrator** | UiPath control plane for robots, queues, attachments, and storage |
| **Storage Bucket** | An Orchestrator-managed file store with signed-URL access |
| **External App** | An OAuth client registered in UiPath for app-to-platform calls; supports PKCE |
| **Unattended Robot** | A robot that runs without operator input on a scheduled or triggered basis |
| **Validation Station** | A Document Understanding surface for human validation of low-confidence extracted fields |
| **Maestro task** | A unit of work within a Maestro process; can be an agent invocation, a robot job, or an Action Center task |

---

*End of document.*
