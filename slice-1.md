# Slice 1 — Onboarding and Discovery Foundations

**Engineering increment:** Cycle 1, October 12–25, 2026  
**Status:** Refinement scope agreed; implementation to start mock-first  
**Implementing project:** New Evergreen State Machine Service. Main API `/onboarding` and GitHub Actions live in other codebases.

## Outcome

> Demonstrate that the new service can onboard managed component/environment records, capture running-image observations, consume and normalize real external signals, and maintain a base-image catalog — **without implementing lifecycle decisions yet**.

This is a service foundation, **not** a production-ready full-estate state machine or a working maintenance-cycle scheduler.

## Refinement scope — preserve these deliverables

| Refined work item | Service involvement | Done means |
|---|---|---|
| Finalize `evergreen.yml` | Consume a validated onboarding representation; coordinate needed fields | Spec supports component-scoped production/nonproduction entries with stable Harness coordinates and nested production validation targets. |
| GitHub Actions workflow calls platform onboarding endpoint | External dependency, not written by state-service agent unless separately tasked | GHA calls existing main API's `/onboarding`. |
| `/onboarding` initializes state and forwards to existing workflow | Main API emits onboarding event; state service is durable initial-state owner | Existing workflow remains intact; component/environment records created on state-service side. |
| Ingest one **real** base-image source | Source adapter owned by state service | One verified feed/API/source connected by end of slice; mock adapter first. |
| Normalize and send base-image internal event | In-process translation | Source-specific payload maps to typed Evergreen event and handler. |
| Base-image store creation/updates | MongoDB | `baseImages` idempotent upserts. |
| Consume live deployment events | Kafka adapter | Real events consumed with source-specific validation/filtering; mock-first. |
| Filter and send internal deployment event | In-process translation | Firm-wide noise filtered; managed environment matched, event normalized. |
| Read live environment for running-image information | Async discovery reader | Given declared Harness coordinates, a real live read can retrieve current application-image facts; mock reader first. |

**Boundary:** implementation within this service should be independently testable even if the separate GHA/API changes are not yet deployed.

## Core use cases

### U1 — New component onboards

1. Receive a `ComponentOnboarded` event from the main API (mock event in initial development).
2. Idempotently upsert `components` and the **top-level** managed production/non-production entries in `environments`.
3. Production `validationEnvironments` remain nested configuration; do **not** create their own environment documents.
4. Initialize **only new** managed environments in lifecycle `DISCOVERY`, with durable initial-discovery pending intent.
5. Acknowledge Kafka input only after durable registration/discovery intent.
6. Asynchronous discovery worker claims and reads environments; persist discovered image facts without overwriting a newer deployment event.
7. Repeated onboarding and configuration-only edits must not erase existing observations or reset state.

### U2 — Completed deployment event arrives

1. Consume from external Kafka source; reject irrelevant event type/non-completed deployment.
2. Source adapter extracts component ID, correlatable Harness coordinates, actual deployed image facts, and deployment **completion** timestamp.
3. Normalize to internal `DeploymentCompleted`; find only an **onboarded** environment by compound identity.
4. Atomically persist if newer than the currently accepted completed deployment; duplicate/stale events are no-ops.
5. No new environment is created for a firm-wide deployment outside Evergreen enrollment; don't query live state for every completed event.
6. Preserve lifecycle state (still `DISCOVERY` in Slice 1).

### U3 — New/changed base image arrives

1. Consume or retrieve an actual selected base-image source; initially test against provisional external-event fixtures.
2. Source adapter maps upstream metadata into an internal `BaseImageDiscovered` / changed event.
3. Upsert the catalog record idempotently; preserve immutable identity and source information where available.
4. Do **not** select a maintenance target or update every environment's lifecycle state.

## Mock-first implementation stages

Stages are **implementation order**, not separate two-week commitments.

| Stage | Deliverables | Proof |
|---|---|---|
| 0. Inspect / plan | Read docs, inspect repository standards; map source and identity gaps | Plan approved; no premature implementation. |
| 1. Service foundation | Bootstrap app; Kafka/MongoDB wiring, three collection repositories, indexes, tests | Can run test MongoDB/Kafka and insert/read records. |
| 2. Onboarding | Provisional mock Kafka message → adapter → typed internal event → handler → component/environment upserts | Lightspeed sample registers PROD and independently managed DEV, validation UAT embedded. |
| 3. Deployment | Mock completed-deployment payload → filter → normalize → conditional environment update | Correct managed record updated; unknown, duplicate, older and conflicting inputs handled. |
| 4. Base-image catalog | Mock source payload → source-specific adapter → normalized event → upsert | Replays do not create duplicate image records. |
| 5. Initial discovery | Durable pending discovery + leased worker + swappable mock/live reader | Async image observation succeeds; failed read retries; multi-replica claim; race does not clobber event. |
| 6. Real integration | One real base-image source, real deployment Kafka stream, and real environment read | Verified field mapping and data; error/lag/health visibility. |

Stages 2–5 can be adjusted to fit existing test infrastructure, but keep them as **small, reviewable changes**.

## Definition of done / suggested acceptance tests

### Onboarding

- Given a new component, creation yields one `components` record and one `environments` record per top-level independently managed deployment.
- Production `validationEnvironments` stay embedded, including multiple targets; no separately managed UAT record is created **solely from the validation list**.
- Replaying the same onboarding event is idempotent.
- Updating a schedule/pipeline/config revision updates configuration but **does not clear** running-image observation, change lifecycle state, or requeue completed initial discovery.
- Partially failed onboarding processing is retryable; pending discovery cannot be lost on Kafka offset acknowledgment.

### Deployment event ingestion

- Valid completed event with matching component and Harness coordinates updates only that environment.
- Unmanaged firm-wide event is rejected/no-op (not an implicit enrollment).
- Same event delivered twice produces no additional material state change.
- An event with earlier completed-deployment timestamp cannot regress the observation.
- Equal timestamps carrying different image identities do not silently win based on arrival order.
- A Kafka processing failure before MongoDB persistence does not cause offset advancement.
- Event transformations can be unit-tested using multiple mock external formats that map to the **same internal type**.

### Initial live discovery

- Worker acquires a pending item with a durable lease; two replicas cannot both successfully claim the same item at once.
- Successful read populates observation; transient failure persists retry status/error summary.
- Process crash while a lease is held is recoverable after lease expiration.
- A delayed first live read cannot overwrite a newer completed-deployment event already stored.
- If the reader cannot provide authoritative base-image provenance, observation records the known application image and leaves its base provenance unknown — never inferred.

### Base-image ingestion

- First source observation inserts a record; a repeated event/upsert remains idempotent.
- Changed metadata is updated without creating a different record for the same immutable identity.
- Catalog updates do not trigger maintenance target selection or lifecycle transitions.
- A failed source poll or feed outage is visible, not interpreted as "no new images".

### Operational behavior

- Multi-replica deployment is supported through Kafka consumer groups and MongoDB atomicity.
- Per-source input counts, filtered counts, errors/retries, consumer lag, and pending discovery counts can be observed (using repository-standard instrumentation).
- No real endpoint credentials are committed; configuration is externalized.

## Explicitly **not** part of Slice 1

- Lifecycle rules/transitions from `DISCOVERY` into `UP_TO_DATE`, `REPAVE_NEEDED`, `BLOCKED`, etc.
- Durable state-evaluation queue or evaluation algorithm; selected target comparison.
- Image selection for the next operational cycle, policy, deferral, debt, RTO sequencing, capacity admission or Temporal scheduler.
- Execution/validation orchestration, evidence collection, production change control, rollback implementation.
- Internal normalized Kafka topics, separate worker microservices, full-estate scheduled rereads, or an advanced state-machine framework.
- Complete base-provenance resolution **unless** a real source already supplies reliable data.
- New APIs for downstream schedulers/operational dashboards (future direction is Kafka state-change notifications + state query API).

## Highest-risk external dependencies

1. **Verified external deployment payload:** confirmed completion semantics, component identifier and correlation to `harnessStage + harnessEnv`; timestamp is actual deployment completion.
2. **Live environment reader:** actual access method and consistency of image identity / deployment revision; race with in-flight Kafka events.
3. **Base-image source:** one usable real publisher/API, family/version/digest identity and catalog metadata.
4. **Onboarding event emission:** main API handoff and reliable event publication alongside its existing workflow; relevant configuration revision semantics.

Mock these boundaries until authoritative samples are available. Document any mismatch, don't retrofit invented assumptions into domain handlers.

## First Codex task — planning only

Copy the following into Codex after installing all four documents in the target repository:

```text
Implement Slice 1 of the Evergreen State Machine Service, but DO NOT WRITE CODE YET.

1. Read AGENTS.md and docs/evergreen/{architecture,slice-1,contracts}.md.
2. Inspect the repository and list reusable framework, Kafka, MongoDB, testing,
   deployment, and observability patterns.
3. Repeat back the confirmed architecture and explicitly distinguish it from
   proposed schema fields and unverified external event/API contracts.
4. Identify missing integration fields and any conflicts with the actual repo.
5. Propose small implementation stages and acceptance tests, starting with
   mocked external events routed through real adapters.
6. Identify the first PR's exact files and tests.

Do not implement lifecycle evaluation, scheduling, policy or Temporal orchestration.
Do not invent production endpoints or event contracts. Wait for approval.
```

After plan approval, the **first small coding task** should typically be the new service bootstrap + MongoDB indexes/repositories + mock-onboarding vertical slice, adapted to the actual repo's language and conventions.
