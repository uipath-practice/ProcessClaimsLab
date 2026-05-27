# 2-data-format — Schemas

This folder is the single source of truth for every JSON shape that flows through the Property Claims Lab. Coding agents read these schemas when generating agents, the case entity, the apps, and the integration plumbing.

## Files

| File | What it defines |
|---|---|
| `envelope.md` | The **unified agent output envelope**. Every agent emits this shape. Centerpiece of the design. |
| `document-fields.md` | The three document-derived JSON shapes: `ClaimData`, `PolicyData`, `AssessmentReportData`. These are extracted at intake / by Agent 01 / by Agent 02 and consumed by every downstream agent. |
| `case-entity.md` | The `PropertyClaimCase` Data Service / Data Fabric entity schema. Every field stored on a case. |
| `action-app.md` | Input payload and output contract for the **Action App** (`claim-review`). The 2-way `continue` / `deny` contract with `reviewerNotes`. |
| `details-inquiry.md` | The `out_DetailsInquiryJSON` shape for the Details Inquiry secondary stage. |
| `payment-instruction.md` | The Finance Payout Queue message payload. |
| `normalisation.md` | Date, currency, address, boolean normalisation rules applied between extraction and persistence. |

## Reading order

If you are new to the project, read the files in this order:

1. `envelope.md` — understand the shared output shape first.
2. `document-fields.md` — see what intake produces.
3. `case-entity.md` — see where everything lands.
4. `action-app.md` — see how the human reviewer plugs in.
5. The remaining files cover narrower contracts.

## Naming conventions

| Surface | Convention | Example |
|---|---|---|
| Envelope JSON output | `out_<AgentName>JSON` | `out_ClaimEligibilityJSON` |
| Auxiliary extracted data | `out_<Topic>JSON` | `out_ClaimDataJSON`, `out_PolicyDataJSON` |
| Maestro input variable | `in_<PascalCase>` | `in_PolicyPDF`, `in_ClaimIXPDataJSON` |
| Case entity field | camelCase / PascalCase per Data Service column | `claimId`, `currentStage` |
| Check rule ID (envelope `checks[].id`) | `<Domain>-<N>` | `E-1`, `C-3`, `P-7`, `CR-2`, `D-1` |
| Date in agent JSON | `DD/MM/YYYY` | `27/05/2026` |
| Date on case entity | ISO 8601 | `2026-05-27` |
| Money amount | bare number; currency in sibling field or context | `12500.00`, currency `USD` |

`AgentName` is PascalCase derived from the agent slug (`claim_eligibility` → `ClaimEligibility`). See `CONFIG.md` §11 for the full agent slug table.

## Architectural decisions baked into these schemas (2026-05-27)

These resolve the open simplification questions surfaced during data-format design. They are binding for downstream specs.

| ADR | Decision | Why |
|---|---|---|
| **DF-1** | Drop `out_<AgentName>Summary` as a separate Maestro variable. The plain-text summary lives inside the envelope as `summary`. Apps, downstream agents, and email templates read `out_<AgentName>JSON.summary`. | Removes 7 redundant variables. Single source of truth for each agent's summary. |
| **DF-2** | Drop `out_isComplete` (Agent 01). | Unused by any downstream agent or app in the locked PDD. |
| **DF-3** | Keep `out_isEligible` (Agent 01) and `out_FinalDecision` (Agent 06) as separate top-level scalars **in addition to** the envelope. | Maestro stage routing is cleaner against a top-level boolean / enum scalar than against `envelope.recommendation` via expressions. |
| **DF-4** | Keep `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_AssessmentReportJSON` as separate top-level variables. | Each is consumed by 4–6 downstream agents. Folding them into `envelope.details` would force every downstream agent to read deep paths. |
| **DF-5** | `details` block is **free-form per agent**. Each agent's `details` contract is documented in that agent's `3-agents/NN_<slug>_agent_payloads.md`. No discriminated-union schema at the envelope level. | Lab simplicity. Each agent documents its own internals. Apps render through "Inspect JSON" or optional per-agent React panels. |
| **DF-6** | Uniform `checks[]` array on every envelope, with stable rule IDs (`E-1`, `C-3`, etc.) and uniform per-check structure. | The whole point of the unified envelope — one app component renders any agent's checks identically. |
| **DF-7** | No absolute monetary thresholds anywhere in agent rules. Escalations are ratio-based or rule-qualitative. | CLAUDE.md locked decision 3. Removes the prior `net_payout > 25,000` rule from Agent 06. |
| **DF-8** | Action App contract is **2-way**: `continue` or `deny`, with free-text `reviewerNotes`. No third "request more info" option in v1. | CLAUDE.md locked decision 5. Reviewer can force-continue or force-deny anything. |
| **DF-9** | `reviewerNotes` from any override is **propagated as input to every downstream agent** in the same case. Agents treat the human decision as authoritative and must not re-raise resolved concerns. | CONFIG.md §10. Reviewer overrides must not be undone by re-analysing the same concern downstream. |
| **DF-10** | Document routing: Claim Form → IXP; Policy → Agent 01 directly (PDF in, JSON out); Incident Report → Agent 02 directly. | CONFIG.md §8. The Policy and Incident Report are reasoned over by the same agents that already do the contextual analysis — no separate IXP step. |

## Variable footprint after simplifications

Per claim, the following Maestro / case variables exist (final count):

| Variable | Producer | Type | Notes |
|---|---|---|---|
| `in_ClaimIXPDataJSON` | IXP (Claim Form) | object | Raw extraction with confidence |
| `in_PolicyID` | Intake robot | string | From claim form |
| `in_PolicyPDF` | Intake robot | job-attachment | Policy file |
| `in_AssessorReportPDF` | Assessor intake robot | job-attachment | Optional at start |
| `out_ClaimEligibilityJSON` | Agent 01 | envelope | Eligibility |
| `out_ClaimDataJSON` | Agent 01 | object | Cleaned claim data |
| `out_PolicyDataJSON` | Agent 01 | object | Policy as JSON |
| `out_isEligible` | Agent 01 | boolean | Maestro routing scalar |
| `out_AssessmentValidationJSON` | Agent 02 | envelope | Report validation |
| `out_AssessmentReportJSON` | Agent 02 | object | Assessment data as JSON |
| `out_CoverageAnalysisJSON` | Agent 03 | envelope | Coverage |
| `out_PayoutCalculationJSON` | Agent 04 | envelope | Payout |
| `out_CredibilityAssessmentJSON` | Agent 05 | envelope | Credibility |
| `out_DecisionJSON` | Agent 06 | envelope | Decision |
| `out_FinalDecision` | Agent 06 | string enum | Maestro routing scalar (approve/partial_approve/deny/escalate) |
| `out_ClaimResponseJSON` | Agent 07 | envelope + letter | Letter and metadata |
| `out_EscalationDecision_Eligibility` | Action App (eligibility) | string enum | continue / deny |
| `out_EscalationComment_Eligibility` | Action App (eligibility) | string | Reviewer notes |
| `out_EscalationDecision_Decision` | Action App (decision) | string enum | continue / deny |
| `out_EscalationComment_Decision` | Action App (decision) | string | Reviewer notes |
| `out_DetailsInquiryJSON` | Inquiry robot | object | Details Inquiry context |

20 variables. Lean enough for a workshop session.

---

*End of README.*
