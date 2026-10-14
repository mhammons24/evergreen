# Evergreen — repository instructions for coding agents

## Before changing code

1. Read `docs/evergreen/architecture.md`, `docs/evergreen/slice-1.md`, and `docs/evergreen/contracts.md`.
2. Inspect the actual repository, build system, language, frameworks, existing APIs, Kafka clients, MongoDB infrastructure, and test conventions. **Do not assume the example file layouts or JSON in the documents already exist.**
3. For the first Codex task, produce a plan and questions **without editing code**. Start implementation only after the engineer approves the plan.
4. If repository behavior conflicts with these design notes, surface the conflict; do not silently overwrite either one.

## Non-negotiable Slice 1 boundaries

- Build the **new Evergreen State Machine Service foundation**, not the entire target-state platform.
- The new service owns its operational-state database with **three domain collections**: `components`, `environments`, `baseImages`.
- A managed environment is uniquely identified within a component by `componentId + harnessStage + harnessEnv`. Each managed environment has its **own** document.
- Only top-level `production` and `nonProduction` entries register independently managed environments. A production entry's `validationEnvironments` are **embedded configuration**, not separate lifecycle entities. Multiple validation targets may exist; all must eventually pass during production execution (not implemented in Slice 1).
- The main Evergreen API owns `/onboarding` and continues the existing workflow. It **publishes an onboarding event** for this service; do not make the main API a second writer of state-service collections.
- The service owns **external source adapters**: consume/filter/validate/translate source-specific messages to **typed, in-process Evergreen events**. No additional internal Kafka topic is required in Slice 1.
- Initial discovery is **asynchronous**: create each environment in lifecycle `DISCOVERY`, persist pending worker intent, and have a background worker read live image/base-image facts. Successful discovery requests **asynchronous state evaluation**. **Only after evaluation transitions the environment out of `DISCOVERY`** do completed-deployment events directly maintain its observations. Do not buffer/replay deployment events for environments in `DISCOVERY` in Slice 1.
- Kafka: at-least-once consumption, commit after durable handling, idempotent writes, stale-event protection using **deployment-completion time**, not Kafka receipt time.
- One deployable service, with multiple replicas. Keep per-source consumers configurable; discovery work must be safely claimable across replicas.
- Discovery changes **observed facts**. Slice 1 also implements a **small, logically separate, asynchronously triggered state evaluator** that transitions `DISCOVERY` to `UP_TO_DATE` or `REPAVE_NEEDED`, and reevaluates monitored environments after observation or catalog changes. Latest available comparable base image is selected **within the existing supported major line**, ordered by highest numeric version then highest numeric build number. Do not perform major-version upgrades. Tag/build semantics beyond verifiable numeric fields are deferred. Do **not** implement deferrals, governance/policy eligibility rules, selected cycle targets, or maintenance scheduling. A published/available catalog entry is a candidate unless the source explicitly marks it unusable.

## Implementation discipline

- **Mock external messages first**, passing them through actual source adapters/handlers; don't test only internal event objects. Mark all mock source contracts as provisional until validated against real payloads.
- Mock the live environment reader behind an interface, then substitute the real reader. **Never invent production endpoints, credentials, event fields, or base-image provenance.**
- Preserve observations and lifecycle state when updating onboarding configuration. Do not full-document replace an environment as a shortcut.
- While `lifecycle.state == DISCOVERY`, the deployment-monitoring handler skips updates for that environment. Initial live discovery establishes observation; the evaluator moves it to `UP_TO_DATE` or `REPAVE_NEEDED` when comparison data is sufficient. Failures/incomplete metadata keep it in `DISCOVERY` with a reason.
- Event handlers do **not** synchronously run the evaluator. Mark durable, per-environment evaluation work after fact changes; a separate worker reads current persisted facts and writes a guarded result. New relevant base-image data must also request bounded reevaluation (not a whole-estate scan inside the Kafka handler).
- A completed-deployment event for an unknown/unmanaged environment must not auto-register it.
- Apply deployment-completion timestamps to reject older monitored deployment updates. Do not allow concurrent evaluator results to overwrite newer observations or assessments. Equal timestamps with conflicting data require explicit handling. During `DISCOVERY`, deployment events are intentionally skipped (no buffering in Slice 1); initial discovery must query live state.
- Include unit tests for adapters and domain handlers, plus integration tests for MongoDB conditional updates, Kafka retry/ack behavior, and a multi-replica discovery claim where feasible.
- Keep changes small and reviewable. Explain assumptions and open questions in the PR or task response.

## Out of scope unless explicitly requested

Full execution-state machine (beyond `DISCOVERY`, `UP_TO_DATE`, `REPAVE_NEEDED`); maintenance-cycle target selection; four-week cycle orchestration; Temporal scheduling; RTO waves; capacity admission; maintenance debt and deferrals; policy enforcement; production validation execution; replacing existing Temporal workflows; broad enterprise migration/refactoring; new standalone microservices.

## Source of truth

- `architecture.md`: confirmed decisions and future context.
- `slice-1.md`: implementation scope, stages, tests, exclusions.
- `contracts.md`: **proposed** minimal documents/events/operations, including intentionally unresolved external fields.

When a choice is marked **OPEN** or **PROPOSED**, ask the engineer or supply alternatives; do not treat it as a confirmed requirement.

**Revision (2026-10-14):** The evaluator and `DISCOVERY`-gated deployment monitoring are now explicitly **in** Slice 1. Older handoff notes saying the evaluator is deferred are superseded.
