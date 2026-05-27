# Action Center tasks

Two task types are created by the Maestro process. Both bind to the same Action App (`claim-review`) but carry different payloads — distinguished by `triggerStage`.

This file defines: when each task is created, how the input payload is assembled, what the output contract is, how Maestro maps the output back to case fields, and how a script can complete a task **without the Action App being deployed** (the path used during Step 2 testing).

---

## 1. Task taxonomy

| Trigger | Created at stage | Created when | Bound app |
|---|---|---|---|
| `eligibility` | `EligibilityAnalysis` | `out_ClaimEligibilityJSON.recommendation == "escalate"` | `claim-review` |
| `decision`    | `DataAnalysis`        | `out_FinalDecision == "escalate"`                       | `claim-review` |

A case can carry at most **one** task at a time. The eligibility-trigger task fires before Data Analysis runs; the decision-trigger task fires after Data Analysis. They are sequential, never concurrent.

---

## 2. Task creation — eligibility trigger

When Eligibility Agent returns `recommendation = "escalate"`, Maestro:

1. Sets `case.triggerStage = "eligibility"`.
2. Sets `case.reviewRequired = true`.
3. Sets `case.status = "escalated"`.
4. Calls Orchestrator Action Center API to create a task.

### Task creation activity

```
ActionCenter.CreateTask(
  appName       = "claim-review",
  taskTitle     = "Eligibility review — " + case.claimId,
  taskPriority  = "medium",
  taskAssignee  = "senior_claims_adjuster",      // group, not individual
  taskInput     = build_eligibility_task_payload(case),
  taskMetadata  = {
    "claimId":       case.claimId,
    "policyId":      case.policyId,
    "triggerStage":  "eligibility"
  }
)
```

### Payload assembly — eligibility

```jsonc
{
  // routing
  "claimId":      case.claimId,
  "policyId":     case.policyId,
  "triggerStage": "eligibility",

  // case header
  "claimantName":     case.claimantName,
  "claimantEmail":    case.claimantEmail,
  "currency":         case.currency,
  "incidentType":     case.incidentType,
  "incidentDate":     case.incidentDate,         // ISO 8601
  "dateOfSubmission": case.dateOfSubmission,     // ISO 8601
  "totalClaimAmount": case.totalClaimAmount,

  // agent outputs available at this trigger
  "out_ClaimDataJSON":             case.out_ClaimDataJSON,
  "out_PolicyDataJSON":            case.out_PolicyDataJSON,
  "out_AssessmentReportJSON":      null,
  "out_ClaimEligibilityJSON":      case.out_ClaimEligibilityJSON,
  "out_AssessmentValidationJSON":  null,
  "out_CoverageAnalysisJSON":      null,
  "out_PayoutCalculationJSON":     null,
  "out_CredibilityAssessmentJSON": null,
  "out_DecisionJSON":              null,

  // attachments — short-lived signed URLs
  "attachments": {
    "claimForm":        StorageBucket.GetReadUri("Claims",                       case.claimFormPdf.StoragePath, 6h),
    "policy":           StorageBucket.GetReadUri("Insurance Policies Repository", case.policyPdf.StoragePath,   6h),
    "assessmentReport": null
  },

  // prior claims
  "priorClaims": case.priorClaims
}
```

Notes:
- The `null` values are emitted explicitly so the Action App's panel-presence check has a uniform shape.
- Signed URLs (`GetReadUri`) live 6 hours per default. The Action App refreshes them on demand if a reviewer opens a preview after expiry.

---

## 3. Task creation — decision trigger

When Decision Agent returns `out_FinalDecision = "escalate"`, Maestro:

1. Sets `case.triggerStage = "decision"`.
2. Sets `case.reviewRequired = true`.
3. Sets `case.status = "escalated"`.
4. Creates the task.

### Payload assembly — decision

Same shape as the eligibility payload, except every agent envelope and the assessment report are populated:

```jsonc
{
  // routing
  "claimId":      case.claimId,
  "policyId":     case.policyId,
  "triggerStage": "decision",

  // case header (same as eligibility payload)
  ...

  // agent outputs — now all present
  "out_ClaimDataJSON":             case.out_ClaimDataJSON,
  "out_PolicyDataJSON":            case.out_PolicyDataJSON,
  "out_AssessmentReportJSON":      case.out_AssessmentReportJSON,
  "out_ClaimEligibilityJSON":      case.out_ClaimEligibilityJSON,
  "out_AssessmentValidationJSON":  case.out_AssessmentValidationJSON,
  "out_CoverageAnalysisJSON":      case.out_CoverageAnalysisJSON,
  "out_PayoutCalculationJSON":     case.out_PayoutCalculationJSON,
  "out_CredibilityAssessmentJSON": case.out_CredibilityAssessmentJSON,
  "out_DecisionJSON":              case.out_DecisionJSON,

  // attachments — assessment report now available
  "attachments": {
    "claimForm":        StorageBucket.GetReadUri("Claims",                       case.claimFormPdf.StoragePath,         6h),
    "policy":           StorageBucket.GetReadUri("Insurance Policies Repository", case.policyPdf.StoragePath,            6h),
    "assessmentReport": StorageBucket.GetReadUri("Assessor Reports",              case.assessmentReportPdf.StoragePath, 6h)
  },

  // prior claims
  "priorClaims": case.priorClaims
}
```

---

## 4. Task output contract

The Action App (or the testing script) submits this on completion:

```jsonc
{
  "decision":      "continue" | "deny",
  "reviewerNotes": "<≥ 20 chars>",
  "reviewedAt":    "<ISO 8601 timestamp with tz>"
}
```

Validation rules at the Action Center level:

| Field | Rule |
|---|---|
| `decision` | Required. Exactly one of `continue` or `deny`. |
| `reviewerNotes` | Required. Minimum 20 characters. Maximum 4000 characters. |
| `reviewedAt` | Server-generated by Action Center when the submit API is called. |

The Action App enforces the 20-char minimum client-side; the testing script must respect it too.

---

## 5. Maestro mapping after task completion

Maestro waits in `ClaimReview` until the task is completed. On completion, it reads `task.decision`, `task.reviewerNotes`, `task.reviewedAt`, and `case.triggerStage`, then writes:

### When `triggerStage = "eligibility"`

```
case.out_EscalationDecision_Eligibility    = task.decision
case.out_EscalationComment_Eligibility     = task.reviewerNotes
case.out_EscalationReviewedAt_Eligibility  = task.reviewedAt
case.triggerStage                          = null
case.reviewRequired                        = false
```

### When `triggerStage = "decision"`

```
case.out_EscalationDecision_Decision    = task.decision
case.out_EscalationComment_Decision     = task.reviewerNotes
case.out_EscalationReviewedAt_Decision  = task.reviewedAt
case.triggerStage                       = null
case.reviewRequired                     = false
```

Maestro then branches per `stages.md` → `ClaimReview` exit expression.

---

## 6. Completing a task **without** the Action App (Step 2 path)

The Orchestrator Action Center API exposes endpoints for listing and completing tasks. A test script can use them in place of the Action App.

### 6.1 List open tasks

```http
GET /orchestrator_/odata/Tasks?
    $filter=Status eq 'Pending' and FolderId eq <folderId>
    &$expand=ApplicationName

Response 200:
{
  "value": [
    {
      "Id":                 12345,
      "Title":              "Eligibility review — CLM-2026-357861",
      "Status":             "Pending",
      "ApplicationName":    "claim-review",
      "CreationTime":       "2026-05-27T11:14:00Z",
      "Priority":           "medium",
      "TaskCatalogName":    "Claim Review",
      "ExternalTag":        null,
      "Tags":               [],
      "Data": {
        "claimId":       "CLM-2026-357861",
        "policyId":      "HO-5534416561",
        "triggerStage":  "eligibility",
        ...
      }
    }
  ]
}
```

### 6.2 Read the task payload

The full payload is on `Task.Data`. The script reads it, decides what to do, and submits a completion.

### 6.3 Complete the task

```http
POST /orchestrator_/odata/Tasks/UiPath.Server.Configuration.OData.CompleteTask
Content-Type: application/json

{
  "taskId": 12345,
  "data": {
    "decision":      "continue",
    "reviewerNotes": "Late filing accepted on documented hospitalisation grounds. Proceed with full analysis.",
    "reviewedAt":    "2026-05-27T11:14:30Z"
  }
}

Response 204: (task closed)
```

The `data` payload here is exactly what the Action App would have submitted. Maestro picks up the completion, unblocks the wait state, and proceeds.

### 6.4 Auth

The script authenticates with the same External App used by the Coded Apps. The OAuth client credentials flow with these scopes is sufficient:

```
OR.Tasks.Write          # Required to complete tasks
OR.Folders              # Folder context
```

`OR.Tasks.Write` is not in the default Coded-Apps scope list (the Action App receives an embedded session). For the test-script path, register a separate External App with `OR.Tasks.Write` or extend the existing one.

### 6.5 A minimal "fake reviewer" script (Python)

```python
#!/usr/bin/env python3
"""
fake-reviewer.py — completes UiPath Action Center tasks for the Property Claims Lab.

Usage:
    python fake-reviewer.py --decision continue --notes "Accepted on hospitalisation grounds. Proceed." [--claim CLM-2026-357861]

Polls the Orchestrator for pending tasks bound to the 'claim-review' app and completes
the matching one(s). For Step 2 testing without the Action App.
"""

import argparse, os, sys, time, requests, json

ORCH = os.environ["UIPATH_ORCH_URL"]                # e.g. https://cloud.uipath.com/<account>/<tenant>
TOKEN = os.environ["UIPATH_ACCESS_TOKEN"]           # OAuth access token with OR.Tasks.Write
FOLDER_ID = int(os.environ["UIPATH_FOLDER_ID"])

H = {
    "Authorization":             f"Bearer {TOKEN}",
    "X-UIPATH-OrganizationUnitId": str(FOLDER_ID),
    "Content-Type":              "application/json"
}

def list_pending():
    r = requests.get(f"{ORCH}/orchestrator_/odata/Tasks",
                     params={"$filter": "Status eq 'Pending' and ApplicationName eq 'claim-review'"},
                     headers=H, timeout=30)
    r.raise_for_status()
    return r.json()["value"]

def complete(task_id, decision, notes):
    body = {
        "taskId": task_id,
        "data": {
            "decision":      decision,
            "reviewerNotes": notes,
            "reviewedAt":    time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        }
    }
    r = requests.post(f"{ORCH}/orchestrator_/odata/Tasks/UiPath.Server.Configuration.OData.CompleteTask",
                      headers=H, json=body, timeout=30)
    r.raise_for_status()

def main():
    p = argparse.ArgumentParser()
    p.add_argument("--decision", choices=["continue", "deny"], required=True)
    p.add_argument("--notes",    required=True)
    p.add_argument("--claim",    help="Optional claimId filter; complete only this case's task")
    a = p.parse_args()
    if len(a.notes) < 20:
        sys.exit("reviewerNotes must be ≥ 20 chars")

    tasks = list_pending()
    if a.claim:
        tasks = [t for t in tasks if t.get("Data", {}).get("claimId") == a.claim]
    if not tasks:
        print("No matching pending tasks.")
        return
    for t in tasks:
        print(f"Completing task {t['Id']} for claim {t['Data']['claimId']} "
              f"({t['Data']['triggerStage']}) → {a.decision}")
        complete(t["Id"], a.decision, a.notes)

if __name__ == "__main__":
    main()
```

Operate it like:

```bash
export UIPATH_ORCH_URL=https://cloud.uipath.com/myaccount/apexmutual-dev
export UIPATH_ACCESS_TOKEN=...
export UIPATH_FOLDER_ID=12345

# Continue past an eligibility escalation
python fake-reviewer.py --claim CLM-2026-357861 --decision continue \
  --notes "Late filing accepted — claimant was hospitalised between 21/02 and 23/02 (documented). Proceed with full analysis."

# Continue past a decision escalation
python fake-reviewer.py --claim CLM-2026-357861 --decision continue \
  --notes "Ratio of 1.314 acceptable given assessor's note on local Bengaluru material prices. Approve at the calculated amount."

# Deny at decision stage
python fake-reviewer.py --claim CLM-2026-357862 --decision deny \
  --notes "Coverage analysis is correct; loss falls under Section II Exclusion 1 (Flood). Deny."
```

The script is the contract for "what the apps would have done". When the Action App is later built, it submits the same JSON to the same endpoint and Maestro's downstream behaviour is identical.

---

## 7. Task lifecycle states

| State | Meaning | Maestro behaviour |
|---|---|---|
| `Pending` | Task created, not yet picked up | Waiting in `ClaimReview` |
| `InProgress` | A reviewer has opened the task | Still waiting; UI surfaces the in-progress state |
| `Completed` | Reviewer submitted `decision + reviewerNotes` | Maestro reads `Task.Data`, maps to case, exits `ClaimReview` |
| `Cancelled` | Task cancelled by an admin | Case stays in `ClaimReview` with `reviewRequired = true`; ops takes over |
| `Expired` | Task timed out (5 business days, configurable) | Maestro alerts the ops queue but continues waiting |

---

## 8. Failure scenarios

| Scenario | Behaviour |
|---|---|
| Action Center API unavailable at task creation | Maestro retries 3× with backoff; on persistent failure sets `case.reviewRequired = true` and writes a process-level error event. Case sits at `ClaimReview` until ops intervenes. |
| Action Center API unavailable at completion (Action App perspective) | The Action App or test script surfaces a "Task already completed" or 5xx; reviewer retries. |
| Two reviewers complete the same task | The Action Center API enforces atomicity; only the first completion succeeds. The second receives 409. |
| Maestro restarts mid-wait | On resume, Maestro re-queries the task by ID and continues. Task completion does not require Maestro to be live at the moment of submission. |

---

*End of action-center-tasks.md.*
