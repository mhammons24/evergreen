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
- Initial discovery is **asynchronous**: persist environment records, mark discovery pending, then have a background worker read live image information. Later completed-deployment events supply the necessary observation data for **direct updates**, without a live read for every event.
- Kafka: at-least-once consumption, commit after durable handling, idempotent writes, stale-event protection using **deployment-completion time**, not Kafka receipt time.
- One deployable service, with multiple replicas. Keep per-source consumers configurable; discovery work must be safely claimable across replicas.
- Discovery changes **observed facts**. Lifecycle evaluation is a separate future module. Slice 1 may initialize `DISCOVERY`, but must not calculate `UP_TO_DATE`, `REPAVE_NEEDED`, deferrals, policy, eligibility, or maintenance schedules.

## Implementation discipline

- **Mock external messages first**, passing them through actual source adapters/handlers; don't test only internal event objects. Mark all mock source contracts as provisional until validated against real payloads.
- Mock the live environment reader behind an interface, then substitute the real reader. **Never invent production endpoints, credentials, event fields, or base-image provenance.**
- Preserve observations and lifecycle state when updating onboarding configuration. Do not full-document replace an environment as a shortcut.
- A completed-deployment event for an unknown/unmanaged environment must not auto-register it.
- Do not allow slow initial discovery or an older deployment event to overwrite a newer observed deployment. Equal timestamps with conflicting data require explicit handling.
- Include unit tests for adapters and domain handlers, plus integration tests for MongoDB conditional updates, Kafka retry/ack behavior, and a multi-replica discovery claim where feasible.
- Keep changes small and reviewable. Explain assumptions and open questions in the PR or task response.

## Out of scope unless explicitly requested

State-transition engine; target selection; four-week cycle orchestration; Temporal scheduling; RTO waves; capacity admission; maintenance debt and deferrals; policy enforcement; production validation execution; replacing existing Temporal workflows; broad enterprise migration/refactoring; new standalone microservices.

## Source of truth

- `architecture.md`: confirmed decisions and future context.
- `slice-1.md`: implementation scope, stages, tests, exclusions.
- `contracts.md`: **proposed** minimal documents/events/operations, including intentionally unresolved external fields.

When a choice is marked **OPEN** or **PROPOSED**, ask the engineer or supply alternatives; do not treat it as a confirmed requirement.
