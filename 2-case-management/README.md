# 2-case-management — Case orchestration

This folder defines how the Property Claims Lab is **orchestrated** in UiPath Maestro and persisted in Data Service. It is the bridge between the agent / app contracts (which already exist) and a running deployment.

## Files

| File | What it defines |
|---|---|
| `stages.md` | Per-stage implementation spec: entry conditions, agents/robots invoked, variable wiring, exit conditions, timers, error handling. 5 primary + 2 secondary stages. |
| `maestro-process.md` | The Maestro process structure as a state machine: sequential and parallel blocks, branch expressions, the synchronisation point at the end of Data Analysis, the wait states for Action Center tasks and Details Inquiry. |
| `action-center-tasks.md` | Action Center task definitions for the two escalation points, the payload assembly, the reviewer output mapping, and the Orchestrator API path that lets a script complete a task without the Action App being deployed. |
| `entity-deployment.md` | Data Service `PropertyClaimCase` entity deployment specifics: indexes, permissions per role, retention, idempotent provisioning. References `2-data-format/case-entity.md` for the field catalogue. |
| `testing-without-apps.md` | End-to-end test plan for the no-app phase: scenarios, fake-reviewer scripts, expected case states, debugging tips. |

## Where the model already lives

These files **complement** the work that's already done; they don't duplicate it.

| Concern | Already defined in |
|---|---|
| Business behaviour per stage | `0-process-definition/pdd.md` |
| High-level component map per stage | `1-solution-definition/sdd.md` §5 |
| Lifecycle stages + transition matrix | `1-solution-definition/sdd.md` §7, `2-data-format/case-entity.md` §6 |
| Case entity field catalogue + validation rules | `2-data-format/case-entity.md` |
| Agent inputs / outputs / contracts | `3-agents/` |
| App contracts | `2-data-format/action-app.md` |
| Tech defaults (models, env, connectors, retries) | `CONFIG.md` |

This folder adds the **orchestration mechanics**: Maestro variables, expression syntax, robot job specifications, Action Center task assembly, entity provisioning details, and the test path.

## Design principles for orchestration

1. **The case entity is the only shared state.** Every agent and every robot reads from and writes to `PropertyClaimCase`. There is no cross-process variable passing outside Maestro's scope.
2. **One Maestro variable per agent envelope.** No flattening into the case entity inside Maestro; the envelope lands on the entity as-is.
3. **Stage transitions are explicit.** Every stage has a single, well-defined exit expression. Maestro does not "fall through" or implicitly transition.
4. **Routing scalars exist for branching.** `out_isEligible` and `out_FinalDecision` are the two scalars Maestro branches on; everything else is read for context, not for routing.
5. **Action Center tasks are creatable without the Action App.** The Action App is one front-end for completing a task; the Orchestrator API is another. Both must work.
6. **All timers are configurable.** Default values are in `CONFIG.md` §12; the Maestro process exposes them as configuration variables, not constants.
7. **Reviewer notes are non-destructive.** When a reviewer continues past escalation, the agent envelopes that fired the escalation are kept intact on the case. The reviewer decision is layered on top, never overwriting.
8. **Robots are idempotent.** The intake robot, the settlement robot, and the fake-reviewer script can all be re-run without producing duplicate side effects (emails, queue messages, case-state changes).

## Reading order

If you are about to deploy this:

1. **`stages.md`** — get the stage-by-stage shape clear in your head.
2. **`entity-deployment.md`** — provision the Data Service entity first; everything else depends on it.
3. **`maestro-process.md`** — build the Maestro process.
4. **`action-center-tasks.md`** — wire the Action Center task creation activities.
5. **`testing-without-apps.md`** — run end-to-end with synthetic claims using the fake-reviewer script.

---

*End of README.*
