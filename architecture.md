# Evergreen State Machine Service — architecture decisions

**Status:** Design handoff for Slice 1 implementation  
**Prepared:** 2026-10-14  
**Primary goal:** Create the operational-facts foundation of a new state-machine service while preserving existing Evergreen execution.

> **Reading rule:** This document distinguishes **LOCKED decisions**, **PROPOSED implementation tactics**, and **OPEN integration details**. Do not treat proposed examples as verified interfaces.

## 1. Purpose and boundary

Evergreen already has working rebase/repave execution workflows. This effort introduces a new service that maintains durable knowledge of onboarded components, their independently managed environments, currently running application images, and firm-published base images. The future lifecycle state evaluator will use those facts to determine maintenance needs.

**The service owns:**

- Kafka source connectivity/consumption for onboarding, deployment, and base-image events; source-specific filtering, validation, and external-to-internal translation.
- Typed, **in-process** internal Evergreen event dispatch/handlers.
- Current-state persistence for `components`, `environments`, and `baseImages`.
- Asynchronous initial discovery of images running in newly registered environments.
- Future lifecycle state evaluation, logically distinct from ingestion (not built in Slice 1).

**Outside this service:**

- Repository GitHub Actions onboarding workflow.
- Existing main Evergreen API endpoint `/onboarding`, its request validation, and its forwarding to the current workflow. It emits an onboarding event to this service.
- Existing Temporal rebase/repave/validation execution workflows.
- Future maintenance target selection, governance, policy, deferrals, scheduler, wave admission and capacity controls.

The main API is **not** a second direct writer of this service's operational-state collections.

## 2. Agreed high-level flow

```text
GitHub Actions -> existing main API /onboarding -> existing execution path
                                            |
                                            +-> Kafka onboarding event

External Kafka deployment feed -----+      +-----------------------------+
External base-image source/feed ----+----> | State Machine Service       |
API onboarding Kafka event ---------+      |                             |
                                            | Source-specific adapters    |
                                            |   -> typed internal events  |
                                            |   -> domain handlers        |
                                            |   -> MongoDB                |
                                            |                             |
                                            | Background discovery worker |
                                            |   -> live environment read  |
                                            |   -> observed-facts update  |
                                            +-----------------------------+

MongoDB owned by service: components | environments | baseImages

FUTURE (not Slice 1): observed-fact changes -> asynchronous lifecycle evaluation
                       -> Kafka state-change notifications + authoritative query API
```

Source adapters and domain handlers have different responsibilities. A new external publisher/schema requires an adapter or mapper change, **not** a rewrite of environment-update logic. The internal event boundary is logical/in-process in Slice 1, not a new Kafka topic.

## 3. Locked architecture decision log

| ID | Area | Decision | Implication |
|---|---|---|---|
| AD-01 | Deployment topology | **One** State Machine Service deployment, horizontally replicated | Source consumers can be configured independently; replicas share Kafka partitions within their consumer groups. |
| AD-02 | Data ownership | State Machine Service is sole owner/writer of its operational-state database | Main API emits onboarding event rather than writing the service's three collections. |
| AD-03 | Collections | Three domain collections: `components`, `environments`, `baseImages` | Do not introduce extra domain entities/collections without reason. Reliability infrastructure may need separate treatment later. |
| AD-04 | Lifecycle granularity | Each independently managed environment has its **own** document and lifecycle state | No single component-wide lifecycle state. |
| AD-05 | Identity | Managed environment key: `componentId + harnessStage + harnessEnv` | Enforce a compound unique index; keep Harness coordinates on records. |
| AD-06 | Production validation | `validationEnvironments` are an **embedded property** of a production environment | Do not register validation-only targets as independent managed environments. Multiple targets are supported; all must pass in the future production workflow. |
| AD-07 | Event-source abstraction | Service owns external ingest, filter, and normalization | Replace/add external event source without changing the internal handler. |
| AD-08 | Internal event routing | Typed Evergreen events handled **in-process** | No extra internal Kafka topics for Slice 1. |
| AD-09 | Onboarding | Main API publishes onboarding event; state service initializes records | Environment begins in `DISCOVERY`; preserve current execution path in main API. |
| AD-10 | First observations | Initial environment-image reads run asynchronously after onboarding | Persist pending discovery before acknowledging onboarding; worker can recover after pod loss. |
| AD-11 | Subsequent observations | Completed-deployment events carry data needed to update environment observation directly | No per-deployment live read is required; the deployment completion timestamp is used to reject stale observations. |
| AD-12 | Kafka | At-least-once; acknowledge/commit after durable handling | Idempotency, recoverability, malformed-message handling are required. |
| AD-13 | State evaluation | Evaluation is logically separate from ingestion, inside same future service | Slice 1 stores facts; it does not yet calculate state transitions. |
| AD-14 | Image catalog vs maintenance target | New available images update `baseImages`; future currency compares to **selected/pinned maintenance target**, not automatically latest discovered image | No automatic fleet reevaluation or target selection on new image arrival in Slice 1. |
| AD-15 | Future external interface | Kafka state-change notifications **and** query API preferred | Direction only; not a Slice 1 deliverable. |

## 4. Environment modeling nuances

The submitted `evergreen.yml` defines **the environments Evergreen is allowed to manage**. It does not supply their running application-image identity or image's underlying base-image provenance. Example fields visible in the user's current specimen:

- `apiVersion: v1`, `kind: Evergreen`, `spec.harnessOrganization`
- `spec.environments.production[]` and `spec.environments.nonProduction[]`
- `harnessStage`, `harnessEnv`, `harnessPipeline`, `harnessInfra`, `strategy`, `type`, and optionally `schedule`
- `production[].validationEnvironments[]` with Harness target coordinates

Within **one component**, `harnessStage + harnessEnv` denotes the managed environment's uniqueness. `harnessPipeline`, infrastructure, maintenance timing, strategy and automation type are configuration, **not identity**. A configuration revision should not reset observation or lifecycle state. A change to either identity coordinate is a new identity unless an explicit migration mechanism is later designed.

The component identifier is supplied by onboarding context/main API; it is **not shown** in the submitted YAML excerpt. The Kafka deployment events **will contain a component identifier**. The rest of the actual external event mapping must still be verified.

**Validation target nuance:** A production environment can declare multiple validation target configurations. Each is a target for testing the same rebased production artifact; all must eventually pass before production repave. They are **not** separate state-machine environment documents merely because they appear in `validationEnvironments`. Do not duplicate validation-only UAT/test as a top-level `nonProduction` entry. A rule for handling an accidental dual declaration is proposed but not yet approved.

## 5. Discovery and change semantics

### Initial onboarding / discovery

1. Receive onboarding event from main API, with authoritative component ID and declared managed environments.
2. Persist/update component and environment configuration idempotently. New environments get `lifecycle.state = DISCOVERY` and a **durable pending-discovery marker**.
3. Commit onboarding Kafka offset only once the component/environment initialization and discovery intent are durable.
4. Worker(s) safely claim pending environment(s), call a live source for the running application image, and write observed image facts without overwriting a newer deployment event.
5. Discovery errors stay visible/retryable; they do not automatically imply lifecycle `BLOCKED`.

### Subsequent deployment events

1. Source adapter filters and normalizes a **completed deployment**, not merely an attempt/start.
2. Correlate to an onboarded managed environment by component and environment coordinates.
3. Directly apply the event's image facts, using an atomic conditional update based on **deployment-completion timestamp**, when reliably supplied.
4. Older and duplicate events do not regress the record; equal-timestamp conflicts require a defined handling path.
5. No automatic environment creation from unrecognized firm-wide events.

### Base-image discovery

A replaceable source adapter ingests one real available-image source (its mechanics may be Kafka or a poller, depending on actual source support), normalizes it to an internal image event, and upserts the base-image catalog. Source availability does **not** mean an image is a selected maintenance target. Provenance of a running application image is a separate lookup/problem; **do not infer it from tags or layers without an authoritative source**.

## 6. Runtime and correctness principles

- **Kafka:** consumer groups for each source; processing partitions are distributed across replicas. Partition order alone cannot establish cross-partition environment chronology.
- **MongoDB:** compound unique environment index, atomic environment-level conditional updates, idempotent upserts; no read–modify–replace of entire documents for independent event updates.
- **Async discovery:** a durable claim/lease with expiry and retry state in `environments` is the initial proposed mechanism. Do not acknowledge onboarding while pending work exists only in volatile process memory.
- **Observation ordering:** the deployment-completion timestamp is the expected event ordering signal, assuming semantics/clock reliability are verified. Live reads may not carry comparable timestamps; until verified, an initial read must not overwrite an event-backed observation.
- **Failure handling:** distinguish ignored-unmanaged event, duplicate/stale event, malformed/unprocessable event, transient integration failure, and incomplete discovery. Kafka offset management must not turn a transient failure into message loss.
- **Observability:** source consumer lag; per-source accept/drop/error counts; discovery pending/in-progress/failures; conditional-update stale discards; last successful base-source sync.
- **Security:** use existing organizational Kafka, MongoDB, and live-source access patterns; never store credentials in events, test fixtures or repository files.

Exact retry counts, DLQ setup, lease TTL, indexes beyond identity, and monitoring technology are **implementation proposals to confirm with the repository and platform standards**, not decisions made in the discussion.

## 7. Larger Evergreen direction — context only, **not Scope 1**

Evergreen's future operating model is based on a **four-week maintenance cycle**: **Week 1** estate-wide CI rebases against a selected/pinned base image; **Weeks 2 and 3** capacity-bounded CD repaves through planned LOB cohorts and RTO waves (higher RTO/lower criticality first); **Week 4** buffer and recovery. Each independently managed environment may run a different application image. Rebase creates an updated artifact from that environment's running application code; repave deploys it. For production, the rebased production artifact is first tested in all designated validation environments; the **same artifact** is then used for production, subject to change controls and maintenance windows.

Future lifecycle states include `DISCOVERY`, `UP_TO_DATE`, `REPAVE_NEEDED`, `BLOCKED`, `DEFERRED`, and execution states. Policy/governance, maintenance debt, target selection, Temporal scheduling and capacity admission are **future**. Do not create them in Slice 1 merely because they appear in the platform vision.

Four **two-week engineering delivery increments** (Oct 12–Dec 6, 2026) are a different timeline from the recurring four-week operational maintenance cycle. This handoff addresses only the first delivery increment.

## 8. Remaining open decisions (do not guess)

1. Actual language/framework, existing service conventions, Kafka and MongoDB libraries, runtime/deployment tools.
2. Main API's precise component-ID source, ordering/version semantics for onboarding updates, and publication/recovery mechanism.
3. Verified external Kafka event payloads, topic names, consumer-group configuration and filtering semantics; exact mapping to Harness stage/env.
4. Live environment reader: target system, credentials/access pattern, actual image identity and timestamp/revision supplied.
5. How original application-image base provenance is determined when not supplied by the event or live reader.
6. Base-image source: API/topic/repository/scraper choice, immutable identity and source-supported availability/approval metadata.
7. Operational policy for malformed events, identity collisions, same-timestamp conflicting deployments, discovery lease expiry and retries.
8. What a changed/deleted environment declaration means (retirement/removal vs preserved historical record).

See `contracts.md` for intentionally provisional shapes and `slice-1.md` for implementation stages and acceptance tests.
