# CONFIG.md — Project-level technical configuration

This file holds the technical defaults that coding agents (Claude Code with UiPath skills) use when generating agents, workflows, apps, and case-management artefacts for the Property Claims Lab. When a downstream spec is silent on a technical value, fall back to this file.

The business process is in `0-process-definition/pdd.md`. The solution architecture is in `1-solution-definition/sdd.md`. This file pins the *technical knobs* shared across all of them.

---

## 1. LLM models

Two named model roles. Concrete model IDs are workshop defaults and are environment-overridable through Agent Builder's deployment configuration.

| Role               | Model ID            | Rationale                                                                       |
| ------------------ | ------------------- | ------------------------------------------------------------------------------- |
| `default_model`    | `claude-sonnet-4-6` | Balanced cost and quality for rule-based or template-driven agents.             |
| `reasoning_model`  | `claude-opus-4-7`   | Stronger reasoning for HO-3 interpretation, narrative analysis, and synthesis.  |

### Agent-to-model assignment

| Agent                              | Slug                       | Model              |
| ---------------------------------- | -------------------------- | ------------------ |
| 01 Claim Eligibility Agent         | `claim_eligibility`        | `default_model`    |
| 02 Assessment Validation Agent     | `assessment_validation`    | `default_model`    |
| 03 Coverage Analysis Agent         | `coverage_analysis`        | `reasoning_model`  |
| 04 Payout Calculation Agent        | `payout_calculation`       | `default_model`    |
| 05 Credibility Assessment Agent    | `credibility_assessment`   | `reasoning_model`  |
| 06 Decision Agent                  | `decision`                 | `reasoning_model`  |
| 07 Claim Response Agent            | `claim_response`           | `default_model`    |

When generating an agent, read the assignment from this table. Specs in `3-agents/` reference `default_model` / `reasoning_model` symbolically; do not hard-code model IDs in agent prompts.

---

## 2. Environments

| Environment | Tenant slug          | Purpose                                                                          |
| ----------- | -------------------- | -------------------------------------------------------------------------------- |
| Development | `apexmutual-dev`     | Iterative development; mock data; synthetic-document generator                   |
| Test        | `apexmutual-test`    | End-to-end testing against the synthetic-document generator and stub integrations |
| Production  | `apexmutual-prod`    | Live claims processing                                                           |

Each environment is a separate UiPath tenant under the Apex Mutual organisation. Environment-specific values (URLs, client IDs, bucket names, email sender) are loaded at runtime via the apps' config bootstrap. Do not hard-code them in code or specs.

---

## 3. External App (OAuth)

A single External App is registered per environment for both Coded Apps. OAuth 2.0 with PKCE; no client secret.

Required scopes:

```
OR.Folders                  # Storage Bucket reads via Orchestrator
OR.Administration.Read      # Tenant metadata
DataFabric.Data.Read        # PropertyClaimCase reads
DataFabric.Schema.Read      # Entity discovery
```

Redirect URIs include the deployed app URLs plus `http://localhost:5173` for local development.

---

## 4. Email connector

Channel: **Gmail**, via the UiPath Integration Service Gmail connector.

| Setting           | Value                                                       |
| ----------------- | ----------------------------------------------------------- |
| Sender (Dev/Test) | One mock inbox per environment (no external delivery)       |
| Sender (Prod)     | `claims@apexmutual.example`                                 |
| Locale            | Derived from the claimant's address country (PDD §11)       |
| Templates         | Stored alongside agent / robot artefacts                    |

In Dev and Test, all outbound mail is routed to the environment mock inbox so the full flow can be exercised without contacting real claimants.

---

## 5. Date and number formats

| Surface                                | Format                | Example       |
| -------------------------------------- | --------------------- | ------------- |
| Case entity (Data Fabric)              | ISO 8601 (`YYYY-MM-DD`) | `2026-05-27` |
| Case entity timestamps                 | ISO 8601 with timezone | `2026-05-27T13:42:00Z` |
| Agent JSON inputs / outputs            | `DD/MM/YYYY`          | `27/05/2026`  |
| Source documents                       | Whatever the generator emits | — |

The synthetic-document generator currently emits `DD/MM/YYYY`. Agents read whatever date format is present in the document; the case-write layer normalises to ISO 8601 before persistence. **The generator may be changed if a future requirement needs a different date format** — agents must remain format-tolerant.

Number formatting: agents emit amounts as bare numbers with a separate currency code; they never emit currency symbols inside numeric fields.

---

## 6. Currency

- Multi-currency from the document generator: one currency per claim, set by the policy.
- **No FX conversion** anywhere in the pipeline.
- Currency code (ISO 4217) is stored on the case header as `currency`.
- All amounts within a claim are in that currency. Agents never compare or sum across currencies.

---

## 7. Storage Buckets

Three Orchestrator-managed buckets per environment:

| Bucket name                     | Contents                              |
| ------------------------------- | ------------------------------------- |
| `Insurance Policies Repository` | Insurance Policy PDFs                 |
| `Claims`                        | First Notice of Loss / Claim Form PDFs |
| `Assessor Reports`              | Professional Incident Report PDFs     |

Access pattern from agents and robots: UiPath built-in Storage Bucket activities / Orchestrator API. Access pattern from Coded Apps: short-lived signed URLs obtained via `GetReadUri` (Apps consume them through the React PDF viewer).

---

## 8. Document handling

| Document                          | Path to structured data                                                                                                                                                                |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Claim Form (FNOL)**             | UiPath Document Understanding (IXP) → `in_ClaimIXPDataJSON`. Structured-form extraction; the Claim Form has a regular layout.                                                          |
| **Insurance Policy PDF**          | Passed **directly to the 01 Claim Eligibility Agent** as an input PDF. The agent extracts the fields it needs into `out_PolicyDataJSON` and into its `details` / `evidence` blocks.    |
| **Professional Incident Report PDF** | Passed **directly to the 02 Assessment Validation Agent** as an input PDF. The agent extracts the fields it needs into `out_AssessmentReportJSON` and validates against the claim.   |

Rationale: only the Claim Form is regular enough to extract reliably through IXP. The Policy and the Incident Report are reasoned over by agents — the same agents that already perform the contextual analysis downstream depends on. Doing both in one place avoids an extra extraction layer that adds no value.

> **Note for coding agents.** When generating workflows: wire IXP **only** in front of the Claim Form. The Policy and the Incident Report flow as PDF binaries into agents 01 and 02 respectively. The SDD still describes IXP as the conceptual extraction step (read as: "the data must end up in JSON"); in implementation, the agent itself produces that JSON.

---

## 9. Unified agent output envelope

Every agent emits the same outer JSON shape. The full contract lives in `1-solution-definition/sdd.md` §6.3 and is fully specified in `2-data-format/`. Coding agents must enforce this shape on every agent without exception — the apps depend on it.

Field naming inside the envelope:

| Field                                       | Convention                                                                |
| ------------------------------------------- | ------------------------------------------------------------------------- |
| Envelope JSON output                        | `out_<AgentName>JSON`     (e.g. `out_ClaimEligibilityJSON`)               |
| Plain-text summary output                   | `out_<AgentName>Summary`  (e.g. `out_ClaimEligibilitySummary`)            |
| Agent-specific auxiliary outputs            | `out_<Topic>JSON`         (e.g. `out_ClaimDataJSON`, `out_PolicyDataJSON`) |

`AgentName` is PascalCase derived from the agent slug: `claim_eligibility` → `ClaimEligibility`.

---

## 10. Reviewer notes propagation

When the Senior Claims Adjuster overrides at either escalation point (Eligibility or Decision), the `reviewerNotes` free-text field is captured in the case and **propagated as an input to every downstream agent**. Downstream agents treat the human decision and notes as authoritative: they must not re-raise the concern that the reviewer has already resolved, and they should reference the reviewer's reasoning when their own output would otherwise contradict it.

Agent prompts in `3-agents/` include the directive:

> *If a reviewer override is present in your inputs, treat the reviewer decision as final. Do not re-raise concerns that the reviewer addressed in `reviewerNotes`. Use those notes to inform your own reasoning where they are relevant.*

---

## 11. File and folder naming conventions

| Artefact                        | Convention                                              | Example                                            |
| ------------------------------- | ------------------------------------------------------- | -------------------------------------------------- |
| Agent prompt file               | `3-agents/NN_<agent_slug>_agent_prompts.md`             | `3-agents/01_claim_eligibility_agent_prompts.md`   |
| Agent payload file              | `3-agents/NN_<agent_slug>_agent_payloads.md`            | `3-agents/01_claim_eligibility_agent_payloads.md`  |
| Maestro input variable          | `in_<PascalCase>`                                       | `in_PolicyPDF`, `in_ClaimIXPDataJSON`              |
| Maestro output variable         | `out_<PascalCase>`                                      | `out_ClaimEligibilityJSON`                         |
| Case entity field               | PascalCase / camelCase per the Data Service schema      | `claimId`, `currentStage`                          |
| Process App route name          | `property-claims`                                       | —                                                  |
| Action App route name           | `claim-review`                                          | —                                                  |
| Case entity                     | `PropertyClaimCase` (singular)                          | —                                                  |
| Finance queue                   | `Finance Payout Queue`                                  | —                                                  |

---

## 12. Resilience and retry defaults

| Concern                          | Default                                                                            |
| -------------------------------- | ---------------------------------------------------------------------------------- |
| Agent invocation retries         | 2 retries with exponential backoff (1 s, 4 s)                                       |
| IXP confidence threshold         | 80 % per extracted field; lower → Validation Station                                |
| Action Center task timeout       | 5 business days, then escalate to Claims Operations Manager queue                   |
| Email delivery failure           | Retry queue; case stage does not advance until dispatch is confirmed                |
| Details Inquiry window           | 14 calendar days, reminder at day 7                                                 |

These defaults are baked into Maestro and the unattended robots. They are tuneable per environment.

---

## 13. Auditing

Findings are stored in Data Fabric on the `PropertyClaimCase` entity:

- Every check carries a stable rule ID (E-1, C-3, P-7, CR-2, D-1, etc.).
- Every check carries an `evidence[]` array citing the source document field.
- Reviewer notes (`out_EscalationComment_<Stage>`) are stored alongside agent outputs.

The Process App provides search and filter over these records. No separate audit-export pipeline is built; reconstruction of any decision happens by querying the case in Data Fabric.

---

*End of CONFIG.md.*
