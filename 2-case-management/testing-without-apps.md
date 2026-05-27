# Testing without the apps — Step 2 plan

This file defines how to validate the full lab pipeline **before the Process App and Action App are built**. The objective is to surface payload contracts, envelope mismatches, currency / locale edge cases, and routing bugs while the cost of fixing them is low (one prompt edit + redeploy, no UI regressions to worry about).

The strategy is straightforward: deploy everything except the apps, drive synthetic claims through Maestro, and use a thin Python script (`fake-reviewer.py` — see `action-center-tasks.md` §6.5) to complete Action Center tasks in place of a human in the Action App.

---

## 1. Prerequisites

- Dev tenant ready per `entity-deployment.md` §9 checklist.
- All seven agents published, version-pinned.
- Maestro process `PropertyClaimAdjudication` published.
- Five robot workflows published: Intake, Assessor Intake, Inquiry, Settlement, Notification.
- `fake-reviewer.py` runnable from a workstation with `UIPATH_ORCH_URL`, `UIPATH_ACCESS_TOKEN` (OAuth client credentials with `OR.Tasks.Write`), and `UIPATH_FOLDER_ID` set.
- Synthetic-document generator runnable locally. Generator output: one folder per claim containing claim-form PDF, policy PDF, incident-report PDF (when present), and a manifest JSON (the kind shown in `inputs/document samples/CLM-2026-357861-claim.json`).

A simple `inspect-case.py` script (see §6 below) for reading the Data Service case row over the OData API is helpful but not required.

---

## 2. Test scenarios

Six scenarios cover every branch in the Maestro process. The synthetic-document generator's `Difficulty` and `AppliedDiscrepancies` configuration drives which scenario fires.

### Scenario A — Clean approve (happy path)

| Aspect | Value |
|---|---|
| Generator config | `Difficulty: easy`, `AppliedDiscrepancies: []` |
| Expected route | Intake → EligibilityAnalysis → AwaitingAssessment → DataAnalysis → SettlementAndClosure → Closed |
| Reviewer involvement | None |
| Agent outcomes | 01 `proceed_to_coverage_analysis`; 02 `proceed_to_parallel_analysis`; 03 `proceed_to_payout`; 04 `approve_payout`; 05 `proceed_to_decision`; 06 `approve`; 07 `letter_drafted` |
| Final state | `status = approved`, `letterEmailMessageId` set, `paymentInstructionRef` set |
| Email sent | Acknowledgement, Eligibility outcome, Inspection scheduled, Inspection report received, Analysis complete, Final decision letter |
| Finance Queue | One message with `decisionPath = automated` |

### Scenario B — Eligibility escalation, reviewer continues

| Aspect | Value |
|---|---|
| Generator config | `AppliedDiscrepancies: ["CLAIMS_LATE_FILING"]` — generator constructs a late-filed claim with an explanation in the narrative |
| Expected route | Intake → EligibilityAnalysis → ClaimReview (`triggerStage="eligibility"`) → DataAnalysis → SettlementAndClosure → Closed |
| Reviewer involvement | `fake-reviewer.py --decision continue --notes "Late filing accepted ..."` |
| Agent outcomes | 01 `escalate`; after reviewer continue, 02..06 run with `in_EscalationDecision_Eligibility = continue`; CR-4 timing is downgraded to `low`; 06 may or may not escalate again depending on other signals |
| Final state | `out_EscalationDecision_Eligibility = "continue"`, `out_EscalationComment_Eligibility` set, status reflects 06's decision |

### Scenario C — Eligibility hard deny

| Aspect | Value |
|---|---|
| Generator config | Generator produces a claim with `payment_status = "Lapsed"` |
| Expected route | Intake → EligibilityAnalysis → SettlementAndClosure → Closed |
| Reviewer involvement | None — eligibility recommendation is `deny`, no reviewer needed |
| Agent outcomes | 01 `deny`; 07 produces a denial letter citing payment status |
| Final state | `status = denied`, `letterEmailMessageId` set, `paymentInstructionRef = null` (no payment for denials), `out_DecisionJSON = null` (06 didn't run) |
| Email sent | Acknowledgement, Eligibility outcome (denied), Final decision letter |

### Scenario D — Decision escalation, reviewer continues

| Aspect | Value |
|---|---|
| Generator config | A claim with elevated `TotalClaimAmount` so the P-8 ratio is > 1.20 |
| Expected route | Intake → EligibilityAnalysis → AwaitingAssessment → DataAnalysis → ClaimReview (`triggerStage="decision"`) → SettlementAndClosure → Closed |
| Reviewer involvement | `fake-reviewer.py --decision continue --notes "Ratio acceptable; approve at calculated amount."` |
| Agent outcomes | 01 `proceed_to_coverage_analysis`; 02 `proceed_to_parallel_analysis`; 03 `proceed_to_payout`; 04 `flag_for_review`; 05 `escalate_to_human`; 06 `escalate`; after reviewer continue, 07 uses `effective_decision_if_continued = "approve"` and drafts an approval letter |
| Final state | `out_EscalationDecision_Decision = "continue"`, `status = approved` |

### Scenario E — Partial approval (some items covered, some excluded)

| Aspect | Value |
|---|---|
| Generator config | A claim with one item attributable to a covered peril and one item attributable to an excluded peril (e.g. some items wind-damage covered + some items flood-damage excluded after the storm) |
| Expected route | Intake → EligibilityAnalysis → AwaitingAssessment → DataAnalysis → SettlementAndClosure → Closed |
| Reviewer involvement | None (unless ratio or credibility also escalates) |
| Agent outcomes | 03 `partial_approval`; 06 `partial_approve`; 07 letter explains approved + denied portions |
| Final state | `status = partial_approved`, `out_FinalDecision = "partial_approve"` |

### Scenario F — Decision escalation, reviewer denies

| Aspect | Value |
|---|---|
| Generator config | Same as Scenario D, or any escalated claim |
| Expected route | Intake → ... → DataAnalysis → ClaimReview → SettlementAndClosure → Closed |
| Reviewer involvement | `fake-reviewer.py --decision deny --notes "Reasonableness ratio too far from estimate; deny."` |
| Agent outcomes | 06 `escalate`; reviewer overrides to deny; 07 drafts a denial letter incorporating the reviewer's reasoning naturally |
| Final state | `status = denied`, `paymentInstructionRef = null`, `out_EscalationDecision_Decision = "deny"` |

Optional further scenarios for thorough coverage:
- **G — Both stages escalate.** Reviewer continues at eligibility, then escalates again at decision, reviewer continues again. Validates the two-review chain.
- **H — Multi-currency.** Run claims in INR, HKD, USD, EUR, GBP. Validates currency derivation and that no FX conversion sneaks in.
- **I — Vacancy clause applies.** Property vacant > 60 days with a vandalism loss. Validates the C-7 critical fail path.
- **J — Reject report.** Assessor PDF is wrong claim's report. Validates AV-1 critical fail and direct escalation.

---

## 3. Iteration loop

For each scenario, run the loop:

```
1. Generate a synthetic claim packet from the generator with the scenario config.
2. Drop the packet into the Intake inbox (or trigger the Intake Robot directly via Orchestrator).
3. Wait ~2 minutes for the agents to run.
4. Query the case row via OData. Inspect:
     - case.currentStage
     - the produced envelopes' status / recommendation / details
     - any case.reviewRequired flag
5. If an Action Center task is pending, decide whether the scenario expects a reviewer action.
     - Yes: run fake-reviewer.py with the appropriate --decision and --notes.
     - No: log this as an unexpected escalation and investigate.
6. Wait for the case to reach Closed.
7. Inspect:
     - case.status final value
     - case.letterEmailMessageId
     - case.paymentInstructionRef (or absence for denials)
     - out_ClaimResponseJSON.details.letter_body — does it read correctly?
     - AppliedDiscrepancies from the generator manifest — did the agents fire the matching rule IDs?
8. Record pass / fail and any open issues in a scenario log.
```

The generator's `AppliedDiscrepancies` field gives you a check on the rule IDs the agents should have fired:

| Generator discrepancy | Expected rule firing |
|---|---|
| `CLAIMS_LATE_FILING` | E-5 `late_with_justification` |
| `CLAIMS_LAPSED_POLICY` | E-1 critical fail |
| `CLAIMS_IDENTITY_MISMATCH` | E-2 critical fail |
| `CLAIMS_ADDRESS_MISMATCH` | E-3 critical fail |
| `CLAIMS_OUT_OF_PERIOD` | E-4 critical fail |
| `CLAIMS_FLOOD_EXCLUSION` | C-2 critical fail (Section II Exclusion 1) |
| `CLAIMS_INFLATED_CLAIM` | P-8 / CR-2 ratio elevated |
| `CLAIMS_VACANCY` | C-7 critical fail (vacancy + vandalism / theft) |
| `CLAIMS_ASSESSOR_NO_LICENSE` | AV-3 fail |
| `CLAIMS_PERIL_MISMATCH` | AV-6 critical fail |

If a generator discrepancy is configured but the corresponding rule did not fire, the agent prompt likely needs tightening — or the discrepancy itself was implemented differently in the generator than the prompt expects. Both are useful findings.

---

## 4. What you're looking for

The Step 2 run is the first time the **whole envelope contract** is exercised. Expect to find:

| Likely issue category | What it looks like | Where to fix |
|---|---|---|
| Field-name drift | Agent emits `out_CoverageAnalysisJSON` with a `summary_text` field that the next agent expects to read as `summary` | Agent prompt or payload schema |
| Currency leakage | Agent 04 reports `net_payout: "₹260,000.00"` (string with symbol) instead of `260000.00` (bare number) | Agent prompt — strip symbols rule |
| Reviewer-notes propagation | CR-4 re-raises a late-filing concern that the reviewer already accepted at eligibility | Agent 05 prompt — propagation rule |
| Status derivation | An envelope reports `status = "ok"` despite a `critical` check fail | Agent prompt — status derivation rule |
| Missing evidence | A check fires with empty `evidence[]` even when source paths are obviously present | Agent prompt — evidence requirement |
| Wrong rule ID | Agent 03 emits a check with id `"coverage_1"` instead of `"C-1"` | Agent prompt — rule ID registry |
| Routing scalar mismatch | `out_FinalDecision = "approve"` but `out_DecisionJSON.details.final_decision = "partial_approve"` | Agent 06 prompt — must match |
| Settlement basis confusion | Settlement Robot reads `out_PayoutCalculationJSON.details.settlement_basis` but the agent emitted it at a different path | Agent 04 prompt or robot wiring |
| Currency derivation | `case.currency` set to `null` because Agent 01 emitted `coverage_summary.currency` in a different field | Agent 01 prompt |
| Letter content leakage | The decision letter mentions "rule CR-2" or "credibility flag" | Agent 07 prompt — no-internal-info rule |

Each issue is one prompt edit + redeploy + rerun. Iterate until the matrix is green.

---

## 5. Triggering DetailsInquiry manually

The v1 release does not auto-generate inquiries (per the PDD). To exercise the DetailsInquiry secondary stage during testing, use a small operator utility script:

```python
#!/usr/bin/env python3
"""trigger-inquiry.py — opens a DetailsInquiry on an existing case."""

import sys, requests, time, os, json

ORCH = os.environ["UIPATH_ORCH_URL"]
H = { "Authorization": f"Bearer {os.environ['UIPATH_ACCESS_TOKEN']}",
      "Content-Type":  "application/json" }

claim_id, origin, *questions = sys.argv[1:]
assert origin in ("Intake", "EligibilityAnalysis", "ClaimReview"), origin

payload = {
  "claim_id":              claim_id,
  "originating_stage":     origin,
  "originating_trigger":   "reviewer_request",
  "requested_information": questions,
  "sent_at":               time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
  "reminder_sent_at":      None,
  "deadline_at":           time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime(time.time() + 14*86400)),
  "response_received_at":  None,
  "response":              None,
  "status":                "open",
  "email_message_id":      None,
  "reminder_message_id":   None
}

# Update the case via Data Service OData; the Maestro process must listen for this update.
# (Exact endpoint depends on the platform's Data Service API; sketch only.)
r = requests.patch(
    f"{ORCH}/dataservice_/api/EntityService/PropertyClaimCase('{claim_id}')",
    headers=H,
    json={ "out_DetailsInquiryJSON": payload, "currentStage": "DetailsInquiry" }
)
r.raise_for_status()
print(f"Opened DetailsInquiry on {claim_id} originating from {origin}; deadline {payload['deadline_at']}.")
```

The Maestro process should expose a "resume from DetailsInquiry" hook that the Inquiry Robot picks up when `out_DetailsInquiryJSON.status` transitions to `responded` or `timed_out`. To simulate a claimant response in testing, run a complementary script that writes the response back into `out_DetailsInquiryJSON.response` and flips `status` to `responded`.

---

## 6. Inspecting case state

A minimal `inspect-case.py` for ad-hoc inspection:

```python
#!/usr/bin/env python3
"""inspect-case.py — pretty-print a PropertyClaimCase row by claimId."""

import sys, os, requests, json

claim_id = sys.argv[1]
H = { "Authorization": f"Bearer {os.environ['UIPATH_ACCESS_TOKEN']}" }
r = requests.get(
    f"{os.environ['UIPATH_ORCH_URL']}/dataservice_/api/EntityService/PropertyClaimCase('{claim_id}')",
    headers=H
)
r.raise_for_status()
row = r.json()

# Surface what the typical test run cares about:
keys = ["claimId", "currentStage", "status", "out_isEligible", "out_FinalDecision",
        "triggerStage", "reviewRequired",
        "out_EscalationDecision_Eligibility", "out_EscalationDecision_Decision",
        "letterEmailMessageId", "paymentInstructionRef", "closedAt"]
print(json.dumps({ k: row.get(k) for k in keys }, indent=2))

# Per-envelope status / recommendation summary
for env_field in ["out_ClaimEligibilityJSON", "out_AssessmentValidationJSON",
                  "out_CoverageAnalysisJSON", "out_PayoutCalculationJSON",
                  "out_CredibilityAssessmentJSON", "out_DecisionJSON",
                  "out_ClaimResponseJSON"]:
    e = row.get(env_field)
    if e:
        print(f"  {env_field}: status={e.get('status')!r} recommendation={e.get('recommendation')!r}")
```

This is enough to see whether the case landed where you expect at each stage.

---

## 7. Definition of done for Step 2

Step 2 is considered complete when **all six scenarios (A–F) plus at least three of the optional scenarios (G, H, I, J) run cleanly end-to-end with no manual intervention beyond `fake-reviewer.py`** and the resulting case rows match the expected final states. The remaining backlog (apps, real reviewer UI) can then be planned against a stable contract.

A short "Step 2 sign-off" document — listing each scenario, its run ID, and pass/fail with notes — is the deliverable at the end of Step 2.

---

*End of testing-without-apps.md.*
