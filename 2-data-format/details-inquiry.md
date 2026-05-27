# `out_DetailsInquiryJSON`

Captures the state of the **Details Inquiry** secondary stage: when more information is needed from the claimant after Intake, Eligibility Analysis, or Claim Review. The case enters a wait state with a 14-day timer (7-day reminder) and returns to the originating stage on receipt of a reply or timer expiry.

Per the PDD, the v1 release does not automatically generate inquiries — the stage exists in the lifecycle and is reachable manually from the Action App's reviewer notes. Automatic inquiry generation is scheduled for a subsequent release. This file defines the structure so the case entity can carry it from day one.

---

## 1. Shape

```jsonc
{
  "claim_id":           "CLM-2026-357861",

  "originating_stage":  "EligibilityAnalysis",      // "Intake" | "EligibilityAnalysis" | "ClaimReview"
  "originating_trigger":"reviewer_request",          // "missing_document" | "reviewer_request"
  "requested_information": [
    "Hospital discharge summary or note confirming dates of hospitalisation between 21/05/2026 and 23/05/2026.",
    "Any correspondence with treating physicians during this period that explains the delay in filing."
  ],

  "sent_at":            "2026-05-27T11:20:00Z",     // ISO 8601 with tz
  "reminder_sent_at":   "2026-06-03T11:20:00Z",     // ISO 8601 with tz, or null if not yet sent
  "deadline_at":        "2026-06-10T11:20:00Z",     // 14 days after sent_at, or null
  "response_received_at": null,                      // ISO 8601 with tz, or null
  "response":           null,                        // string (claimant's reply text) or null

  "status":             "open",                      // "open" | "responded" | "timed_out"

  "email_message_id":   "<message-id>@apexmutual.example",   // Integration Service message handle
  "reminder_message_id": null
}
```

---

## 2. Field reference

| Field | Type | Required | Description |
|---|---|---|---|
| `claim_id` | string | yes | Case key; identical to the case's `claimId`. |
| `originating_stage` | enum | yes | The stage the case will return to once the inquiry closes. |
| `originating_trigger` | enum: `missing_document` \| `reviewer_request` | yes | Why the inquiry was opened. `missing_document` from Intake; `reviewer_request` from Eligibility or Claim Review (typed into the reviewer notes by the Senior Claims Adjuster). |
| `requested_information` | array of string | yes | One bullet per piece of information requested. Used directly to compose the email body. |
| `sent_at` | timestamp | yes | When the request email was sent. |
| `reminder_sent_at` | timestamp \| null | no | When the 7-day reminder email was sent. Null until day 7. |
| `deadline_at` | timestamp | yes | `sent_at + 14 days`. The Maestro timer fires at this point if no response received. |
| `response_received_at` | timestamp \| null | yes | When the claimant replied. Null while the inquiry is open. |
| `response` | string \| null | yes | The claimant's reply text or attachment summary. Null while the inquiry is open. |
| `status` | enum: `open` \| `responded` \| `timed_out` | yes | Drives Maestro's exit path. |
| `email_message_id` | string | yes | Integration Service handle for the request email. Used for retries and replies-correlation. |
| `reminder_message_id` | string \| null | no | Handle for the reminder email. |

---

## 3. Lifecycle

```
[Originating stage] ──(more info needed)──► DetailsInquiry
                                              │
                                              │ sent_at = now()
                                              │ deadline_at = sent_at + 14d
                                              │ status = "open"
                                              ▼
                                          (waiting)
                                              │
                ┌─────────────────────────────┼──────────────────────────────┐
                │                             │                              │
            (day 7)                       (response)                    (day 14)
                │                             │                              │
        reminder_sent_at = now()       response_received_at = now()    status = "timed_out"
        reminder_message_id            response = <text>
                │                       status = "responded"
                │                             │                              │
                └─────────────────────────────┼──────────────────────────────┘
                                              ▼
                                  [Originating stage] (resume)
```

Transitions are driven by Maestro. The case's `currentStage` is `DetailsInquiry` for the entire duration; only the originating stage is recorded in the inquiry object.

---

## 4. Example sequence

The Senior Claims Adjuster, looking at the eligibility escalation in the Action App, decides she wants confirmation of the claimant's hospitalisation dates before continuing the claim. She types her question into `reviewerNotes` and clicks **Continue with note**.

Maestro recognises this as a `reviewer_request` from the eligibility stage and instead of routing forward, creates a `DetailsInquiry` instance:

```jsonc
{
  "claim_id":              "CLM-2026-357861",
  "originating_stage":     "EligibilityAnalysis",
  "originating_trigger":   "reviewer_request",
  "requested_information": [
    "Hospital discharge summary or note confirming dates of hospitalisation between 21/05/2026 and 23/05/2026.",
    "Any correspondence with treating physicians that explains the delay in filing."
  ],
  "sent_at":               "2026-05-27T11:20:00Z",
  "reminder_sent_at":      null,
  "deadline_at":           "2026-06-10T11:20:00Z",
  "response_received_at":  null,
  "response":              null,
  "status":                "open",
  "email_message_id":      "<a1b2c3d4@apexmutual.example>",
  "reminder_message_id":   null
}
```

The robot sends the email immediately. Seven days later, no reply, the reminder fires. Twelve days later, the claimant replies with an attached PDF. The robot updates the inquiry:

```diff
- "response_received_at": null,
+ "response_received_at": "2026-06-08T16:02:00Z",
- "response":             null,
+ "response":             "Claimant attached hospital discharge summary confirming admission 21/05/2026 19:30, discharge 23/05/2026 12:15.",
- "status":               "open",
+ "status":               "responded"
```

Maestro then transitions the case back to `EligibilityAnalysis`. The Senior Claims Adjuster sees the updated context in the Action App, satisfies herself, and clicks **Continue**. The claim proceeds.

---

## 5. v1 behaviour caveat

Automatic inquiry generation is out of scope for the v1 lab. In the v1 flow:

- Intake never auto-generates an inquiry. Missing documents pause at Intake without entering `DetailsInquiry`.
- Eligibility and Claim Review reviewers can type a question into `reviewerNotes`, and the inquiry **may** be created manually via an operations utility — but the Action App itself routes only to `continue` or `deny`.

The full automated workflow lives in v2. Defining the structure now ensures the case entity is forward-compatible without a schema migration.

---

*End of details-inquiry.md.*
