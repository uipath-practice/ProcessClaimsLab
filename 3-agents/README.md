# 3-agents — Agent specifications

This folder contains the prompt + payload pair for every agent in the Property Claims Lab. Coding agents read these files when generating the agents in UiPath Agent Builder.

## Roster

| # | Agent | Slug | Stage | Model role |
|---|---|---|---|---|
| 01 | Claim Eligibility Agent | `claim_eligibility` | 2 — Eligibility Analysis | `default_model` |
| 02 | Assessment Validation Agent | `assessment_validation` | 3 entry | `default_model` |
| 03 | Coverage Analysis Agent | `coverage_analysis` | 3 parallel | `reasoning_model` |
| 04 | Payout Calculation Agent | `payout_calculation` | 3 parallel | `default_model` |
| 05 | Credibility Assessment Agent | `credibility_assessment` | 3 parallel | `reasoning_model` |
| 06 | Decision Agent | `decision` | 3 synthesis | `reasoning_model` |
| 07 | Claim Response Agent | `claim_response` | 5 — Settlement | `default_model` |

Model role IDs resolve through `CONFIG.md` §1. Agent prompts reference roles symbolically; concrete model IDs are not pinned in prompts.

## Files per agent

| File | Purpose |
|---|---|
| `NN_<slug>_agent_prompts.md` | System prompt: Role, Inputs Provided, Instructions, Output Format, Special Rules, User Prompt. |
| `NN_<slug>_agent_payloads.md` | Inputs JSON schema, Output Format narrative, Output JSON schema, Sample payload. |

`<payloads>` placeholders inside the prompts file are replaced at runtime by the corresponding content from the payloads file. Do not expand them in source.

## Contracts every agent obeys

1. **Unified envelope.** Every analytical output conforms to the envelope in `2-data-format/envelope.md`. The shared keys (`status`, `recommendation`, `checks`, `findings`, `summary`, `details`) are present without exception.
2. **Rule IDs.** Each `check.id` is drawn from the registry in `2-data-format/envelope.md` §3. New rules are added by extending the registry, not by inventing IDs.
3. **No absolute monetary thresholds.** Escalation triggers are ratio-based or rule-qualitative. Reference `CLAUDE.md` decision 3.
4. **Reviewer-notes propagation.** When `in_EscalationDecision_*` and `in_EscalationComment_*` are present on inputs, the agent treats the human decision as authoritative and does not re-raise concerns the reviewer has resolved. See `CONFIG.md` §10.
5. **Currency.** Agents never compare or convert amounts across currencies. The currency comes from the policy and stays that way.
6. **Dates.** Agents read whatever date format the source documents carry and emit `DD/MM/YYYY` in agent JSON. ISO 8601 is the case-entity format and is applied by the case-write layer, not by agents.

## Data flow

```
Intake (RPA + IXP for Claim Form only)
  → in_ClaimIXPDataJSON, in_PolicyID, in_PolicyPDF, in_AssessorReportPDF
       │
       ▼
[01] Claim Eligibility Agent
       │  Produces: out_ClaimDataJSON, out_PolicyDataJSON, out_ClaimEligibilityJSON, out_isEligible
       │
       │ (recommendation = proceed_to_coverage_analysis)
       ▼
[02] Assessment Validation Agent
       │  Produces: out_AssessmentReportJSON, out_AssessmentValidationJSON
       │
       ▼  (Maestro fork)
┌────────────────────── parallel block ──────────────────────┐
[03] Coverage Analysis Agent     ──► out_CoverageAnalysisJSON
[04] Payout Calculation Agent    ──► out_PayoutCalculationJSON
[05] Credibility Assessment Agent ──► out_CredibilityAssessmentJSON
└────────────────────────────────────────────────────────────┘
       │  (Maestro join)
       ▼
[06] Decision Agent
       │  Produces: out_DecisionJSON, out_FinalDecision
       │
       │  (escalate → Action Center → reviewer override → reviewerNotes feed-back)
       ▼
[07] Claim Response Agent
       │  Produces: out_ClaimResponseJSON
       ▼
Settlement (RPA)
```

## Decision priority (Agent 06)

**Deny > Escalate > Partial Approve > Approve.**

Escalation always takes priority over approval. Denial always takes priority over escalation.

## Consistency checklist before finalising any agent

- [ ] Every variable name in the prompt's `User Prompt:` section appears in `Inputs JSON`.
- [ ] Output envelope conforms to `2-data-format/envelope.md`.
- [ ] Every check has a stable rule ID from the registry.
- [ ] `status` is derived from check severities (not chosen freely).
- [ ] `recommendation` is from the agent's enum table.
- [ ] No absolute dollar threshold anywhere in the rules.
- [ ] Reviewer-notes directive present in the Role / Special Rules.
- [ ] The agent's `details` shape is documented in the payloads file with a sample.

---

*End of README.*
