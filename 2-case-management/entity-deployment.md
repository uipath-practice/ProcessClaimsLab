# `PropertyClaimCase` entity deployment

The `PropertyClaimCase` Data Service / Data Fabric entity is the system of record for the lab. The field catalogue and validation rules are defined in `../2-data-format/case-entity.md`. This file adds **deployment specifics** that the data-format spec deliberately leaves out: indexes, permissions, retention, environment provisioning, and idempotent migration.

---

## 1. Provisioning order

When deploying to a new tenant (Dev, Test, Prod), provision in this order:

1. **External App** (OAuth) — registered first because every other component authenticates against it.
2. **Storage Buckets** — `Insurance Policies Repository`, `Claims`, `Assessor Reports`. Robots and agents need them before any case can be created.
3. **Data Service entity** `PropertyClaimCase` — see §2 below.
4. **Orchestrator queue** `Finance Payout Queue` — Settlement Robot writes here.
5. **Integration Service connector** Gmail (one mailbox per environment).
6. **Agents** — published to Agent Builder, versions pinned.
7. **Maestro process** — `PropertyClaimAdjudication` published with references to the deployed agents.
8. **Robots** — published Studio workflows: Intake, Assessor Intake, Inquiry, Settlement, Notification.

The Maestro process must not be published before the entity, the agents, and the robots all exist — Maestro fails to deploy a process referencing missing assets.

---

## 2. Entity creation

### 2.1 Field types

The full field-by-field catalogue with types is in `../2-data-format/case-entity.md` §1. Use that as the source of truth when scripting the entity creation in Data Service.

A condensed type summary:

| Group | Data Service column types |
|---|---|
| Identity / header | Text (short), Text (medium), Text (long) for free-form fields |
| Dates | Date (no time) for `incidentDate`, `dateOfSubmission`; DateTime (UTC) for timestamps |
| Currency / amounts | Decimal(18,2) |
| Booleans | Boolean (nullable where the field is nullable per the catalogue) |
| Attachment refs | Text (long) holding the storage bucket URI + filename + MIME — or a custom record type if the platform supports it |
| Agent envelopes (`out_*JSON`) | JSON (text) — Data Service holds them as serialized JSON, opaque to the entity schema |
| Reviewer outputs | Text + Text + DateTime |
| Routing scalars | Text (short) for enums, Boolean for `out_isEligible` |
| `priorClaims[]` | JSON (text) holding the array |

### 2.2 Required vs nullable

- Required at intake (Intake Robot must populate or the row is rejected): `claimId`, `policyId`, `claimantName`, `claimantEmail`, `propertyAddressLine1`, `propertyCity`, `propertyState`, `propertyCountry`, `incidentType`, `incidentDate`, `dateOfSubmission`, `totalClaimAmount`, `claimFormPdf`, `policyPdf`, `currentStage`, `status`, `createdAt`, `updatedAt`.
- All other fields are nullable and populated as the case progresses.

### 2.3 Validation constraints

Translate the rules in `../2-data-format/case-entity.md` §5 to entity-level validation:

| Rule | Implementation |
|---|---|
| **R-CE-1** `claimId` unique | Unique index (also serves as primary lookup key) |
| **R-CE-2** `incidentDate ≤ dateOfSubmission` | Computed-field check or pre-write validation in the Intake Robot. Data Service may not support cross-field constraints natively — enforce in the robot and surface a data-quality flag if violated. |
| **R-CE-3** `triggerStage` consistency | Enforced by the Maestro process. Entity has no constraint. |
| **R-CE-4** approve implies positive payout | Enforced by the Settlement Robot before writing `status = approved`. |
| **R-CE-5** `closedAt` iff `currentStage == Closed` | Enforced by the Settlement Robot. |
| **R-CE-6** `currency == out_PolicyDataJSON.coverage_summary.currency` | Enforced by Agent 01's case-write step. |

Where Data Service supports field-level rules natively (e.g. enum constraints, length limits, regex), use them. Where it doesn't (cross-field), enforce in the producer (robot or agent post-processing).

---

## 3. Indexes

| Index | Fields | Purpose |
|---|---|---|
| **PK** | `claimId` (unique) | Primary lookup |
| `IX_currentStage` | `currentStage` | Process App dashboard tabs by stage |
| `IX_status` | `status` | Process App KPI tiles |
| `IX_policyId` | `policyId` | Prior-claims lookup at intake; cross-claim queries |
| `IX_claimantName_lower` | `lower(claimantName)` | Prior-claims lookup by name (best-effort match) |
| `IX_reviewRequired` | `reviewRequired` (boolean) | Operations queue view ("needs attention") |
| `IX_createdAt` | `createdAt` | Sort by recency |
| `IX_closedAt` | `closedAt` | Reporting / retention sweeps |

If the Data Service platform supports composite indexes, add `(currentStage, status)` for the dashboard.

---

## 4. Permissions

Three principals interact with the entity. Assign least-privilege scopes per principal.

| Principal | Read | Write | Notes |
|---|---|---|---|
| **Intake Robot** | header, attachments | header, attachments, `priorClaims`, `out_ClaimIXPDataJSON` | Cannot write agent or reviewer fields |
| **Assessor Intake Robot** | header, attachments | `assessmentReportPdf`, `reviewRequired`, `currentStage` | |
| **Inquiry Robot** | header | `out_DetailsInquiryJSON`, `currentStage`, `status` | |
| **Settlement Robot** | full row | `out_ClaimResponseJSON`, `letterEmailMessageId`, `paymentInstructionRef`, `status`, `closedAt`, `currentStage` | |
| **Notification Robot** | header, eligibility envelope, decision envelope | (read only — sends emails) | |
| **Agent 01** | `out_ClaimIXPDataJSON`, `policyId`, `claimFormPdf`, `policyPdf` | `out_ClaimEligibilityJSON`, `out_ClaimDataJSON`, `out_PolicyDataJSON`, `out_isEligible`, `currency` | |
| **Agent 02** | claim, policy, eligibility envelope, eligibility reviewer notes, assessment report PDF | `out_AssessmentValidationJSON`, `out_AssessmentReportJSON` | |
| **Agents 03, 04, 05** | claim, policy, assessment, eligibility envelope, assessment validation envelope, eligibility reviewer notes, (05 also) `priorClaims` | their respective envelope only | |
| **Agent 06** | all upstream envelopes, eligibility reviewer notes | `out_DecisionJSON`, `out_FinalDecision` | |
| **Agent 07** | full row | `out_ClaimResponseJSON` | |
| **Process App (read-only)** | full row | none | |
| **Action App** (via Maestro) | task-specific subset of the row | reviewer outputs only, scoped to `triggerStage` | Permissions are enforced by Maestro's task-data plumbing, not by the entity. |

In practice these scopes are configured at the Data Service permission set level, with one permission set per agent / robot / app. The Maestro process invokes each component under its own identity so the scopes apply.

---

## 5. Retention

Apex Mutual's stated retention for property claims is **seven years from `closedAt`**.

Implementation options:

| Approach | Description | Recommended for v1 |
|---|---|---|
| Native Data Service TTL | If the platform exposes row-level TTL per condition, set `closedAt + 7 years` as expiry. | Use if available |
| Periodic sweep | A scheduled robot queries `closedAt < now() - 7y` and archives + deletes. | Fallback |
| Tiered storage | Move closed cases to cold storage after 1 year. | Out of scope for v1 |

Attachments in Storage Buckets follow the same retention. Configure the bucket lifecycle policy to match: 7-year retention from `created_at` of the bucket object.

---

## 6. Idempotent provisioning

The entity provisioning script must be idempotent so it can be re-run during development without manual cleanup. The pattern:

```
1. Check if entity 'PropertyClaimCase' exists in the tenant.
2. If exists, diff field set; add any missing fields; do not drop existing fields.
3. If not exists, create with the full field catalogue.
4. Always re-apply indexes (idempotent in most platforms).
5. Always re-apply permission sets.
```

Schema migrations during the lab (e.g. adding a new agent → new field) should be additive only. Never remove or rename a field without an explicit migration. The downstream agents and apps can tolerate unknown extra fields but not missing ones.

---

## 7. Bucket layout

Each environment has three buckets. Within each bucket, organise by claim ID:

| Bucket | Object key pattern | Example |
|---|---|---|
| `Claims` | `claims/<claimId>/claim-form.pdf` | `claims/CLM-2026-357861/claim-form.pdf` |
| `Insurance Policies Repository` | `policies/<policyId>.pdf` | `policies/HO-5534416561.pdf` |
| `Assessor Reports` | `claims/<claimId>/assessor-report.pdf` | `claims/CLM-2026-357861/assessor-report.pdf` |

Policies are keyed by `policyId` (one PDF per policy, not per claim) so multiple claims against the same policy share the policy file. The other two buckets are claim-scoped.

Bucket-level encryption: at-rest using the tenant's default key. In-transit via HTTPS. Signed URLs (`GetReadUri`) expire after 6 hours by default.

---

## 8. Environment matrix

| Setting | Dev | Test | Prod |
|---|---|---|---|
| Tenant slug | `apexmutual-dev` | `apexmutual-test` | `apexmutual-prod` |
| Entity name | `PropertyClaimCase` (same in all environments) |
| Bucket names | same names; physically separate per tenant |
| Email sender | dev mock inbox | test mock inbox | `claims@apexmutual.example` |
| Finance Queue | stub consumer that logs only | stub consumer that logs only | real consumer (Finance Engineering) |
| Retention | 30 days (Dev), 90 days (Test), 7 years (Prod) |
| Sample data | synthetic claim packets from generator | synthetic + selected anonymised real claims (future) | real claims only |

Per-environment values (tenant URLs, queue IDs, External App client IDs) are loaded at deployment time from `CONFIG.md` and the runtime configuration store; do not hard-code them in the entity definition.

---

## 9. Deployment checklist

Before declaring an environment "ready":

- [ ] External App registered; OAuth client ID + scopes confirmed.
- [ ] Three Storage Buckets exist with correct names and lifecycle policy.
- [ ] `PropertyClaimCase` entity exists with the full field catalogue.
- [ ] All eight indexes created.
- [ ] All permission sets assigned (one per agent / robot / app).
- [ ] `Finance Payout Queue` exists.
- [ ] Gmail Integration Service connector configured with the correct sender mailbox.
- [ ] All seven agents published in Agent Builder, version-pinned.
- [ ] Maestro process `PropertyClaimAdjudication` published.
- [ ] Five robot workflows published (Intake, Assessor Intake, Inquiry, Settlement, Notification).
- [ ] `fake-reviewer.py` script can authenticate and list pending tasks (Dev / Test only).
- [ ] One synthetic claim runs end-to-end through the happy path with no escalation.

---

*End of entity-deployment.md.*
