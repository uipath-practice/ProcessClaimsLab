# Unified agent output envelope

This is the **central design rule** of the lab. Every agent emits this shape. Apps render every agent through one component. Agent-specific structure lives only inside the `details` block — never at the top level.

If you are generating an agent, an app panel, an email template, or anything that reads agent output, this file is the contract.

---

## 1. The envelope at a glance

```jsonc
{
  // ---------- routing & overall status ----------
  "claim_id":       "CLM-2026-357861",        // the case key — must match in every envelope for the same case
  "status":         "ok",                      // "ok" | "warn" | "critical" — one-glance health for tab badges
  "recommendation": "proceed_to_coverage_analysis",  // agent-specific enum (see §4)

  // ---------- the audit trail ----------
  "checks": [                                  // every rule the agent ran, pass or fail; uniform shape across agents
    {
      "id":       "E-1",                       // stable rule ID — see §3
      "label":    "Policy in force",
      "result":   "pass",                      // "pass" | "fail" | "n/a"
      "severity": "info",                      // "info" | "warn" | "critical"
      "detail":   "Payment status: Current — Paid in Full",
      "evidence": [
        { "source": "policy.declarations.payment_status", "value": "Current - Paid in Full" }
      ]
    }
  ],

  // ---------- human-readable bullets and paragraph ----------
  "findings":  [
    "Policy is in force and payment is current.",
    "Claimant identity matches the named insured."
  ],
  "summary":   "All five eligibility checks passed. The claim is eligible to proceed to coverage analysis.",

  // ---------- agent-specific structured payload ----------
  "details":   { /* free-form per agent — documented in 3-agents/NN_<slug>_agent_payloads.md */ }
}
```

The seven envelope keys (`claim_id`, `status`, `recommendation`, `checks`, `findings`, `summary`, `details`) are present on **every** agent output. The shape of `details` is the only thing that varies per agent.

---

## 2. Field reference

| Field | Type | Required | Description |
|---|---|---|---|
| `claim_id` | string | yes | The case key. Same value across every envelope on the case. Format: insurer-defined (e.g. `CLM-2026-357861`). |
| `status` | enum: `ok` \| `warn` \| `critical` | yes | One-glance health. Drives the tab badge in the Apps. Derived from the worst-severity check that failed (none → `ok`; warn → `warn`; any critical fail → `critical`). |
| `recommendation` | string enum | yes | Agent-specific routing signal. See §4 for the enum per agent. |
| `checks` | array of `Check` | yes | Every rule the agent ran. Order is not significant. Empty array is permitted only when the agent is non-analytical (e.g. Agent 07 letter-drafting). |
| `findings` | array of string | yes | 1–5 short bullets, suitable for direct human display in the Apps. May summarise multiple checks. |
| `summary` | string | yes | 2–6 sentence plain-text paragraph. The Apps surface it as the agent's "Notes" card. Emails and downstream agents read it for context. |
| `details` | object | yes | Agent-specific structured payload. Free-form; documented in each agent's `payloads.md`. May be `{}` if the agent has no extra structured output. |

### `Check` object

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Stable rule identifier. Pattern `<Domain>-<N>` where Domain is one of `E` (eligibility), `C` (coverage), `P` (payout), `CR` (credibility), `D` (decision), `AV` (assessment validation), `R` (response). N is a positive integer. |
| `label` | string | yes | Short human label (≤ 60 chars). |
| `result` | enum: `pass` \| `fail` \| `n/a` | yes | The check's outcome. `n/a` means the check did not apply (e.g. flood exclusion when peril is fire). |
| `severity` | enum: `info` \| `warn` \| `critical` | yes | How a `fail` should be presented. `pass` and `n/a` checks set this to `info`. Drives the row colour in the Apps. |
| `detail` | string | yes | One-sentence outcome. ≤ 200 chars. |
| `evidence` | array of `Evidence` | no | Pointers to the source facts the check used. Optional but strongly preferred — empty array (`[]`) is the default. |

### `Evidence` object

```jsonc
{
  "source": "policy.declarations.payment_status",   // dot-path into a known JSON structure
                                                    // or document-citation like "incident_report.cause_determination"
  "value":  "Current - Paid in Full"                // the observed value at that path
}
```

Evidence paths point into one of the canonical JSON structures: `claim` (= `out_ClaimDataJSON`), `policy` (= `out_PolicyDataJSON`), `assessment` (= `out_AssessmentReportJSON`), or the envelope of a prior agent (`eligibility`, `coverage`, etc.).

---

## 3. Rule ID registry

Rule IDs are stable across releases — once published, an ID is never reused for a different concept. Apps and audit logs reference rules by ID.

| Domain | Range | Owner | Examples |
|---|---|---|---|
| `E-1`..`E-5` | Eligibility | Agent 01 | E-1 Policy in force, E-2 Identity match, E-3 Address match, E-4 Coverage period, E-5 Filing deadline |
| `AV-1`..`AV-6` | Assessment validation | Agent 02 | AV-1 Document identity, AV-2 Required fields present, AV-3 Assessor credentials, AV-4 Property / claim match, AV-5 Internal consistency, AV-6 Assessment-claim peril alignment |
| `C-1`..`C-8` | Coverage analysis | Agent 03 | C-1 Peril identified, C-2 Section II exclusions, C-3 Coverage A applies, C-4 Coverage B applies, C-5 Coverage C named peril, C-6 Coverage D applies, C-7 Vacancy clause, C-8 Duty to mitigate |
| `P-1`..`P-9` | Payout calculation | Agent 04 | P-1 Items categorised, P-2 Section A limit, P-3 Section B limit, P-4 Section C limit, P-5 Section C sublimits, P-6 Deductible applied, P-7 Settlement basis (RC vs ACV), P-8 Reasonableness ratio, P-9 Section D limit |
| `CR-1`..`CR-5` | Credibility assessment | Agent 05 | CR-1 Narrative consistency, CR-2 Estimate reasonableness (ratio), CR-3 Documentation completeness, CR-4 Timing and pattern, CR-5 Prior-claims pattern |
| `D-1`..`D-4` | Decision | Agent 06 | D-1 Hard denial gate, D-2 Escalation gate, D-3 Partial vs full approve, D-4 Decision priority |
| `R-1`..`R-3` | Claim response | Agent 07 | R-1 Effective decision selected, R-2 Letter completeness, R-3 Reviewer comment incorporated |

When you add a new rule, allocate the next free ID within its domain and document it in the agent's `payloads.md`. Do not renumber existing rules.

---

## 4. `recommendation` enums per agent

| Agent | `recommendation` enum |
|---|---|
| 01 Claim Eligibility Agent | `proceed_to_coverage_analysis` \| `deny` \| `escalate` |
| 02 Assessment Validation Agent | `proceed_to_parallel_analysis` \| `escalate` \| `reject_report` |
| 03 Coverage Analysis Agent | `proceed_to_payout` \| `deny_all` \| `partial_approval` \| `escalate` |
| 04 Payout Calculation Agent | `approve_payout` \| `flag_for_review` \| `escalate` |
| 05 Credibility Assessment Agent | `proceed_to_decision` \| `escalate_to_human` |
| 06 Decision Agent | `approve` \| `partial_approve` \| `deny` \| `escalate` |
| 07 Claim Response Agent | `letter_drafted` \| `letter_pending_review` |

Decision priority at Agent 06 (CLAUDE.md): **Deny > Escalate > Partial Approve > Approve**.

---

## 5. Status derivation rule

`status` is derived from the worst-severity check that failed, NOT chosen freely:

```
let crit = any check with result="fail" AND severity="critical"
let warn = any check with result="fail" AND severity="warn"
status = crit ? "critical" : warn ? "warn" : "ok"
```

Agents implement this rule rather than picking a status by feel. Apps trust the agent's status field for UI rendering.

---

## 6. Worked example — Agent 01 envelope

Real shape that Agent 01 emits for the prior HK sample claim (CLM-2026-409027):

```json
{
  "claim_id": "CLM-2026-409027",
  "status": "critical",
  "recommendation": "escalate",
  "checks": [
    { "id": "E-1", "label": "Policy in force",  "result": "pass", "severity": "info",
      "detail": "Payment status: Current - Paid in Full",
      "evidence": [{ "source": "policy.declarations.payment_status", "value": "Current - Paid in Full" }] },

    { "id": "E-2", "label": "Identity match",  "result": "pass", "severity": "info",
      "detail": "Claimant name matches policyholder name.",
      "evidence": [
        { "source": "claim.ClaimClaimant[0].Name",                     "value": "Wong Ka Man" },
        { "source": "policy.declarations.named_insured",               "value": "Wong Ka Man" }
      ] },

    { "id": "E-3", "label": "Address match",   "result": "pass", "severity": "info",
      "detail": "Claim and policy addresses match.",
      "evidence": [
        { "source": "claim.ClaimProperty[0]",                          "value": "Flat B, 23/F, Tower 3, The Harbourside, ..." },
        { "source": "policy.declarations.described_property.address",  "value": "Flat B, 23/F, Tower 3, The Harbourside, ..." }
      ] },

    { "id": "E-4", "label": "Coverage period", "result": "fail", "severity": "critical",
      "detail": "Claim submission date (14/03/2026) precedes incident date (10/03/2026).",
      "evidence": [
        { "source": "claim.Claim[0].DateOfSubmission",         "value": "14/03/2026" },
        { "source": "claim.ClaimIncident[0].DateOfIncident",   "value": "10/03/2026" }
      ] },

    { "id": "E-5", "label": "Filing deadline", "result": "n/a",  "severity": "info",
      "detail": "Not evaluated — coverage period check failed first.",
      "evidence": [] }
  ],
  "findings": [
    "Policy and identity check passed.",
    "Coverage period check is critical: claim date precedes incident date — likely a date format misinterpretation."
  ],
  "summary": "Eligibility cannot be determined. The claim submission date (14/03/2026) is earlier than the incident date (10/03/2026), which is logically impossible and suggests a date-format parsing problem in the source documents. All other eligibility checks (policy in force, identity match, address match) passed.",
  "details": {
    "is_eligible_strict": false,
    "missing_documents":  [],
    "policy_currency":    "HKD"
  }
}
```

The corresponding scalar variables Agent 01 also emits:

| Variable | Value |
|---|---|
| `out_isEligible` | `false` |

---

## 7. Worked example — Agent 06 envelope (Decision)

```json
{
  "claim_id": "CLM-2026-357861",
  "status": "warn",
  "recommendation": "escalate",
  "checks": [
    { "id": "D-1", "label": "Hard denial gate",      "result": "pass", "severity": "info",
      "detail": "No hard denial conditions present.", "evidence": [] },
    { "id": "D-2", "label": "Escalation gate",       "result": "fail", "severity": "warn",
      "detail": "Late filing flagged with justification (E-5) — requires human review.",
      "evidence": [{ "source": "eligibility.checks[id=E-5].result", "value": "late_with_justification" }] },
    { "id": "D-3", "label": "Partial vs full",        "result": "n/a",  "severity": "info",
      "detail": "Not evaluated — escalation gate fired first.", "evidence": [] },
    { "id": "D-4", "label": "Decision priority",      "result": "pass", "severity": "info",
      "detail": "Priority order applied: Deny > Escalate > Partial > Approve.", "evidence": [] }
  ],
  "findings": [
    "Decision: escalate to human reviewer.",
    "Driver: late filing with justification noted by Eligibility (E-5)."
  ],
  "summary": "All hard denial conditions are clear and all parallel analyses (coverage, payout, credibility) returned proceed signals. However, the eligibility step flagged a late filing with justification, which is an escalation trigger per the decision rules. The claim is sent for human review with the full analysis attached.",
  "details": {
    "final_decision":       "escalate",
    "denial_reasons":       [],
    "escalation_reasons":   [
      "Late filing with justification (E-5)."
    ],
    "approved_payout_reference": {
      "currency":         "INR",
      "net_payout":       125000.00,
      "settlement_basis": "replacement_cost"
    },
    "confidence_level":     "medium",
    "escalation_flag_count": 1
  }
}
```

Scalar variables Agent 06 also emits:

| Variable | Value |
|---|---|
| `out_FinalDecision` | `"escalate"` |

---

## 8. App contract

A single React component, `GenericAnalysisPanel`, renders any agent's envelope. It reads, in order:

1. `status` → coloured banner (`ok` = green, `warn` = amber, `critical` = red).
2. `recommendation` → pill in the header.
3. `findings[]` → bulleted list.
4. `checks[]` → table with stable rule IDs as a column.
5. `summary` → "Notes" card.
6. `details` → "Inspect JSON" expander, plus an optional agent-specific React panel registered by agent slug.

The same component is used in both the Process App (read-only dashboard) and the Action App (single-claim review). No agent gets a bespoke layout.

---

## 9. Things NOT in the envelope

- Plain-text `out_<AgentName>Summary` is **not** a separate variable (ADR DF-1). The summary lives at `envelope.summary`.
- `out_isEligible` and `out_FinalDecision` are separate top-level scalars for Maestro routing, **not** inside the envelope (ADR DF-3).
- `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_AssessmentReportJSON` are separate top-level variables, **not** inside `details` (ADR DF-4).

---

*End of envelope.md.*
