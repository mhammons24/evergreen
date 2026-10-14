# Slice 1 — Minimal documents, events, evaluation and persistence operations

**Status:** Implementation starting point, **not approved production schemas**  
**Prepared:** 2026-10-14  
**Rule:** Identity and behavior listed as **LOCKED** are design decisions. Example payload field names/values, source-specific mapping, and several operational fields are **PROPOSED** until actual source contracts and repository code are inspected.

This document covers only Slice 1, **including initial and ongoing currency evaluation**. Do not add future policy, scheduling, Temporal workflow history, or execution state fields just because future architecture is known.

**Revision 2026-10-14:** `DISCOVERY` gates deployment monitoring; asynchronous evaluation moves environments to `UP_TO_DATE` / `REPAVE_NEEDED`; latest same-major available/comparable version/build is a temporary target-selection rule.

## 1. Identity and ownership

### LOCKED

- `componentId` is an authoritative **component identifier passed by the main API's onboarding event**; it is not in the shown `evergreen.yml` excerpt.
- A managed environment is unique within a component by `(componentId, harnessStage, harnessEnv)`.
- `harnessOrganization` is given at `spec` level in the current YAML example; it is **not** part of the agreed within-component uniqueness key.
- Only entries under `environments.production[]` and `environments.nonProduction[]` become state-managed environment records.
- Entries under `production[].validationEnvironments[]` are **embedded child configuration on production**; each is a validation target, **not** a separately managed environment record. Multiple configured targets must eventually pass before production proceeds (future workflow concern).
- The state-service database has **exactly three domain collections**: `components`, `environments`, `baseImages`.

### PROPOSED MongoDB constraints

```javascript
// Confirm naming and collation conventions in the actual repository.
db.environments.createIndex(
  { componentId: 1, harnessStage: 1, harnessEnv: 1 },
  { unique: true, name: "unique_component_harness_environment" }
);

db.environments.createIndex(
  { "initialDiscovery.status": 1, "initialDiscovery.nextAttemptAt": 1 },
  { name: "pending_initial_discovery" }
);

// PROPOSED: index for durable asynchronous evaluation work.
db.environments.createIndex(
  { "evaluation.pending": 1, "evaluation.nextAttemptAt": 1 },
  { name: "pending_currency_evaluation" }
);
```

Before enforcing the unique index, decide and test exact input normalization: whether Harness identifiers are case-sensitive, how whitespace is handled, and whether organization scopes need special handling. Do **not** silently lowercase identifiers without source-system assurance.

## 2. Source YAML reference and important correction

The actual user example shows:

```yaml
apiVersion: v1
kind: Evergreen
spec:
  harnessOrganization: wf
  environments:
    production:
      - harnessStage: prod
        harnessEnv: lightspeed-prod
        harnessPipeline: DOE_K8S_Rolling_Deploy
        harnessInfra: ocp-lew09,ocp-man09
        strategy: parallel
        type: Manual
        schedule: "0 0 22 ? * MON"
        validationEnvironments:
          - harnessStage: test
            harnessEnv: lightspeed-uat
            harnessPipeline: DOE_K8S_Rolling_Deploy
            harnessInfra: ocp-lew09,ocp-man09
    nonProduction:
      - harnessStage: dev
        harnessEnv: lightspeed-dev
        harnessPipeline: DOE_K8S_Rolling_Deploy
        harnessInfra: ocp-lew09,ocp-man09
        strategy: parallel
        type: FullyAutomated
```

**Important modification:** The screenshot also included `test/lightspeed-uat` as a top-level non-production entry. The subsequent design decision was to **avoid duplicating a validation-only environment as an independently managed non-production environment**, so it is deliberately absent from the reference above. This is a *desired revised example*, not a verbatim transcription or claim that the current YAML has already been edited.

The main API/event provides `componentId`; don't fabricate it from repository name unless agreed explicitly.

## 3. Minimal documents — PROPOSED

### `components`

```json
{
  "_id": "lightspeed",
  "repository": "some-org/lightspeed",
  "harnessOrganization": "wf",
  "configRevision": "revision-from-main-api",
  "onboardedAt": "2026-10-14T12:00:00Z",
  "updatedAt": "2026-10-14T12:00:00Z"
}
```

**Responsibilities:** immutable component identity, onboarding metadata, shared Harness organization and config revision. Full arrays of independent environment state do not belong here. Repository and revision information are illustrative and depend on what the main API can actually provide. A Git SHA is **not** an ordered revision number; do not compare SHA strings to decide which configuration is newer.

### `environments` — production example

```json
{
  "_id": "opaque-internal-environment-id",
  "componentId": "lightspeed",
  "harnessStage": "prod",
  "harnessEnv": "lightspeed-prod",
  "environmentType": "PRODUCTION",
  "configuration": {
    "harnessPipeline": "DOE_K8S_Rolling_Deploy",
    "harnessInfra": "ocp-lew09,ocp-man09",
    "strategy": "parallel",
    "type": "Manual",
    "schedule": "0 0 22 ? * MON",
    "validationEnvironments": [
      {
        "harnessStage": "test",
        "harnessEnv": "lightspeed-uat",
        "harnessPipeline": "DOE_K8S_Rolling_Deploy",
        "harnessInfra": "ocp-lew09,ocp-man09"
      }
    ]
  },
  "observation": null,
  "initialDiscovery": {
    "status": "PENDING",
    "attempts": 0,
    "nextAttemptAt": "2026-10-14T12:00:00Z",
    "leaseOwner": null,
    "leaseUntil": null,
    "lastError": null
  },
  "lifecycle": { "state": "DISCOVERY" },
  "evaluation": {
    "status": "NOT_EVALUATED",
    "pending": false,
    "requestedRevision": 0,
    "evaluatedRevision": 0,
    "nextAttemptAt": null,
    "leaseUntil": null,
    "reason": null,
    "evaluatedAgainstBaseImageId": null
  }
}
```

**Note:** `initialDiscovery` is **worker retry/lease metadata**, not another business state machine or independent monitoring toggle. The **lifecycle state itself gates monitoring**: deployment updates are ignored in `DISCOVERY`; after live discovery, a separate async evaluator transitions to `UP_TO_DATE` or `REPAVE_NEEDED` when comparison data is sufficient. `evaluation` fields above are a PROPOSED durable-work mechanism; exact names/claim semantics need repo review. Configuration, observation and lifecycle state are updated independently; never replace the entire environment document for a partial update.

### `environments` — observation after a completed deployment

```json
{
  "observation": {
    "applicationImage": {
      "reference": "registry.example/lightspeed:1.9.3",
      "digest": "sha256:example"
    },
    "baseImage": null,
    "completedDeploymentAt": "2026-10-14T12:30:00Z",
    "source": "DEPLOYMENT_EVENT",
    "sourceEventId": "deployment-example-id",
    "recordedAt": "2026-10-14T12:30:03Z"
  }
}
```

This is the **`observation` subdocument**, not a separate collection or entire `environments` record. `baseImage` is nullable: don't assume the live-reader or deployment feed gives authoritative original-base provenance. `recordedAt` is an audit/processing time, **not** an event-ordering timestamp. Event order uses `completedDeploymentAt` when reliable. Store reference/digest only if returned by the real source; do not invent digests.

### `baseImages`

```json
{
  "_id": "source-specific-stable-image-identity",
  "source": "firm-base-image-source",
  "family": "jdk17",
  "version": "17.0.11",
  "buildNumber": 8,
  "majorLine": "jdk17",
  "versionParts": [17, 0, 11],
  "imageReference": "registry.example/base/jdk17:17.0.11",
  "digest": "sha256:example",
  "availability": "AVAILABLE",
  "buildNumber": 8,
  "majorLine": "jdk17",
  "versionParts": [17, 0, 11],
  "publishedAt": "2026-10-14T10:00:00Z",
  "lastObservedAt": "2026-10-14T12:00:00Z"
}
```

**Do not lock `_id`, family/version semantics, digest availability, versionParts, buildNumber, majorLine, or approval values until the actual source is inspected.** The numeric version parts, majorLine and buildNumber are **provisional comparison metadata**, not asserted to be present upstream. A version alone might not uniquely identify an image across registries, architectures or build variants. For Slice 1, choose the latest available/comparable image in the **currently observed supported major line** by numeric version first, then numeric build; do not compare across major versions. A published/available record is considered a candidate unless the source explicitly marks it unusable; do not invent approval gates. Future operational scheduling will supply a pinned target instead.

## 4. Internal Evergreen events — PROPOSED minimal contracts

Internal names are conceptual, not verified organizational message names. They are **typed in-process objects**, not their own Kafka topics in Slice 1.

### `ComponentOnboarded`

```json
{
  "type": "ComponentOnboarded",
  "eventId": "onboard-example-01",
  "componentId": "lightspeed",
  "configRevision": "revision-from-main-api",
  "repository": "some-org/lightspeed",
  "harnessOrganization": "wf",
  "production": [
    {
      "harnessStage": "prod",
      "harnessEnv": "lightspeed-prod",
      "harnessPipeline": "DOE_K8S_Rolling_Deploy",
      "harnessInfra": "ocp-lew09,ocp-man09",
      "strategy": "parallel",
      "type": "Manual",
      "schedule": "0 0 22 ? * MON",
      "validationEnvironments": [
        { "harnessStage": "test", "harnessEnv": "lightspeed-uat", "harnessPipeline": "DOE_K8S_Rolling_Deploy", "harnessInfra": "ocp-lew09,ocp-man09" }
      ]
    }
  ],
  "nonProduction": [
    { "harnessStage": "dev", "harnessEnv": "lightspeed-dev", "harnessPipeline": "DOE_K8S_Rolling_Deploy", "harnessInfra": "ocp-lew09,ocp-man09", "strategy": "parallel", "type": "FullyAutomated" }
  ]
}
```

**Handler:** upsert component; upsert managed environments by unique key; initialize **new** environments in `DISCOVERY` + pending initial discovery. Never reset an existing observation due to onboarding retry or config edit.

### `DeploymentCompleted`

```json
{
  "type": "DeploymentCompleted",
  "eventId": "deployment-example-123",
  "componentId": "lightspeed",
  "harnessStage": "prod",
  "harnessEnv": "lightspeed-prod",
  "applicationImage": {
    "reference": "registry.example/lightspeed:1.9.3",
    "digest": "sha256:example"
  },
  "baseImage": null,
  "completedAt": "2026-10-14T12:30:00Z"
}
```

**Handler:** filter by managed key **and lifecycle state**. While `DISCOVERY`, **skip deployment-event monitoring updates without buffering or replay**; the initial live read is authoritative for bootstrap. Otherwise compare completion timestamp atomically, update observed application image and any *authoritative* supplied base information, and durably request independent reevaluation on a changed observation. No upsert for unknown environment. Equal timestamps with same facts are a no-op; equal timestamps with different facts require explicit conflict behavior.

**Verified:** upstream completed deployment events will have a component identifier and timestamps; the discussion **assumed** timestamp means completion time. **Still verify:** names/availability of `harnessStage`, `harnessEnv`, image digest and original base provenance in the actual event. The adapter may need an explicit mapping if the event has different coordinate properties.

### `BaseImageDiscovered`

```json
{
  "type": "BaseImageDiscovered",
  "eventId": "base-example-2026-10-14",
  "source": "firm-base-image-source",
  "sourceImageId": "source-image-identity",
  "family": "jdk17",
  "version": "17.0.11",
  "buildNumber": 8,
  "majorLine": "jdk17",
  "versionParts": [17, 0, 11],
  "imageReference": "registry.example/base/jdk17:17.0.11",
  "digest": "sha256:example",
  "publishedAt": "2026-10-14T10:00:00Z"
}
```

**Handler:** source-identity-aware idempotent upsert to `baseImages`. Metadata fields depend on the verified upstream source. A meaningful change to the latest available/comparable image requests bounded asynchronous reevaluation for matching managed environment major lines. **Do not synchronously evaluate in the source consumer** and do not initiate maintenance-cycle target selection or execution.

## 5. External mock fixtures — separate from internal contracts

Mock **external** formats so adapters are exercised. For illustration, different vendors can report the same completed deployment using different payload names:

```json
{
  "vendorEventType": "RELEASE_FINISHED",
  "component": "lightspeed",
  "stage": "prod",
  "environment": "lightspeed-prod",
  "deployedArtifact": "registry.example/lightspeed:1.9.3",
  "finishedAt": "2026-10-14T12:30:00Z",
  "result": "SUCCESS"
}
```

```json
{
  "messageKind": "applicationDeploymentChanged",
  "appComponentId": "lightspeed",
  "harnessCoordinates": { "stageId": "prod", "envId": "lightspeed-prod" },
  "image": "registry.example/lightspeed:1.9.3",
  "deploymentCompletedAt": "2026-10-14T12:30:00Z"
}
```

**These two events are wholly invented test fixtures**, not evidence of real Harness or firm-wide event schemas. Both adapters should yield the same domain-level meaning where the source semantics genuinely match. Source adapters should reject unsupported/ambiguous event types rather than guessing.

The real source adapters must be revised against captured authorized examples before being considered integrated.

## 6. Persistence operations and safety properties

### Onboarding upsert

- Upsert `components` by authoritative component ID.
- For each **top-level** managed production/non-production entry, upsert `environments` keyed by `(componentId,harnessStage,harnessEnv)`.
- On **insert only**, set `lifecycle.state = DISCOVERY` and `initialDiscovery.status = PENDING`; other configuration fields may update on subsequent onboarding.
- Never unset/reset observation and lifecycle state when reprocessing the same or newer onboarding configuration.
- If one handler writes multiple documents, make failure recovery idempotent. Transactions may be used if platform standards require atomic visibility; don't assume all-updates-atomic just because one Kafka message triggered them.
- **OPEN:** how to detect/reject stale onboarding config revisions; whether an environment absent from a new config should be retired.

### Deployment conditional update (conceptual MongoDB)

```javascript
// Pseudocode. Exact field paths/concurrency API subject to implementation.
db.environments.updateOne(
  {
    componentId: incoming.componentId,
    harnessStage: incoming.harnessStage,
    harnessEnv: incoming.harnessEnv,
    "lifecycle.state": { $ne: "DISCOVERY" },
    $or: [
      { "observation.completedDeploymentAt": { $lt: incoming.completedAt } },
      { "observation.completedDeploymentAt": { $exists: false } }
    ]
  },
  {
    $set: {
      "observation.applicationImage": incoming.applicationImage,
      // Also set observation.baseImage based on the NEW application image:
      // - trusted incoming base identity when provided;
      // - otherwise unknown/null for a different app image;
      // - optionally preserve verified provenance for the SAME immutable app image.
      // Implement this consistently with the conditional update, not by
      // carrying forward an unrelated older image's base provenance.
      "observation.completedDeploymentAt": incoming.completedAt,
      "observation.source": "DEPLOYMENT_EVENT",
      "observation.sourceEventId": incoming.eventId
    }
  },
  { upsert: false }
);
```

**Caution:** Build the base-image portion of the update deliberately: a newly observed application image must **not inherit** the previous image's base provenance. If the event does not supply authoritative provenance, mark it unknown; an existing verified base identity may be retained only when the immutable application image is demonstrably unchanged. Implement this with an atomic update/pipeline or other concurrency-safe operation. The pseudocode leaves this field out until that rule is implemented.

No match can mean (a) unmanaged/unregistered environment, (b) **still in `DISCOVERY` (intentionally not monitored)**, (c) stale event, or (d) equal timestamp; distinguish these before logging/metrics and handling conflicts. A successful change must also durably request evaluation (e.g., update revision/pending marker atomically with the observation write). Do not apply evaluation results against a superseded observation.

### Initial discovery update

- Worker claims one `PENDING`/retryable environment atomically with lease owner and expiry; respects lease recovery after restart.
- Live reader returns image facts plus **source-provided deployment timestamp or revision if available**.
- While the environment remains `DISCOVERY`, event monitoring is **disabled**; live reading is responsible for establishing the initial observation. No special event buffering or catch-up mechanism for this phase is required.
- Atomically guard worker leases/results so a slower/expired retry cannot overwrite a newer successful initial discovery attempt.
- Once sufficient observation data is persisted, durably request evaluation; remain in `DISCOVERY` until evaluation can establish `UP_TO_DATE` or `REPAVE_NEEDED`. Missing provenance/catalog/comparison inputs remain unresolved with a reason.
- Persist retries/error details without conflating discovery worker status with lifecycle state.

### Base-image catalog upsert

- Resolve stable identity from verified source; idempotently insert/update current metadata.
- Do not invent an approval gate: published/available records are candidates for Slice 1 unless the source explicitly marks them unusable. Respect source-provided flags when verified; their mapping remains OPEN.
- On a meaningful change, request **bounded async reevaluation** of affected environments within supported major line; not an immediate estate-wide scan in a Kafka handler.
- Store per-source synchronization errors/last-success status through suitable instrumentation; exact operational persistence is OPEN.

## 7. Slice 1 currency evaluator contract — LOCKED behavior / PROPOSED mechanics

The evaluator **does not interpret external events**. It reads the current persisted environment observation and candidate base-image catalog data, then produces an assessment. Ingestion merely persists facts and schedules evaluation work. The evaluator is asynchronous and must not be executed inline in Kafka message handling.

**Inputs:** observed original base-image identity (with supported line/major version and numeric version/build where available), latest available comparable catalog base image, current observed-facts revision.
**Output:** proposed state `UP_TO_DATE` or `REPAVE_NEEDED`, target image used, reason, evaluated input revision. When inputs are incomplete/ambiguous, return **UNRESOLVED evaluation outcome** without claiming a fresh lifecycle state; initial environments stay in `DISCOVERY` and monitoring remains inactive. For an already monitored environment, the previous lifecycle state may remain for history but its evaluation status becomes **UNRESOLVED/STALE**, explicitly not safe for future scheduling decisions.

**Temporary target resolver (Slice 1):**

1. Restrict candidates to the environment's **current supported major-version line**. Do not upgrade from e.g. JDK 17 to JDK 21, even if version 21 is numerically greater.
2. Use published/available images as candidates; respect any explicit source-provided exclusion/ineligibility flags. Do not invent an approval workflow or require one to exist for Slice 1.
3. Among valid candidates, compare numeric version components first, then **numeric build number** for equal versions. Do not sort raw tag strings lexicographically. Equal ranks with different immutable images require a tie/ambiguity rule (OPEN).
4. If observed image equals selected comparable target, return `UP_TO_DATE`; if its comparable version/build rank is older, return `REPAVE_NEEDED`. Do not automatically downgrade when observed rank is newer than catalog; return UNRESOLVED pending investigation.
5. Later the four-week scheduler will supply a **pinned cycle target**; keep the evaluator independent of the target-selection implementation.

**Durability / concurrency:** PROPOSED `evaluation.pending`, requested/evaluated revisions, and leased worker on `environments`. Mark pending in the same durable update as each successful observation write. For relevant catalog changes, enumerate/page affected environments in bounded batches and mark evaluation pending; do not attempt a global synchronous transition on a Kafka consumer. Worker reads *latest facts*, and atomically verifies input revision/target before writing lifecycle state. Avoid dropping reevaluation requests on crash. Multiple replicas must not overwrite each other's newer evaluations.

**Monitoring gate:** `lifecycle.state == DISCOVERY` means skip completed-deployment monitoring updates. Once evaluator commits `UP_TO_DATE` or `REPAVE_NEEDED`, subsequent completed-deployment events can update observation and request reevaluation; no independent `MONITORING` state or initialization phase field is needed.

## 8. Contract verification checklist before wiring real sources

- [ ] Main API event contains stable `componentId` and all top-level managed environment coordinates/configuration.
- [ ] Config revision ordering/duplication strategy is explicit.
- [ ] Completed-deployment event's component ID matches the onboarding domain.
- [ ] External deployment event identifies or can deterministically map to `harnessStage` and `harnessEnv`.
- [ ] Event `completedAt` is **deployment completion time**, not Kafka or publication time.
- [ ] Image identity is sufficient to distinguish changes (reference/digest as actually provided).
- [ ] Image's original base-image identity is provided or deliberately modeled as unknown.
- [ ] Base image has sufficient trustworthy major-line, version, build and eligibility metadata to make the provisional comparison; otherwise result is unresolved.
- [ ] Catalog change can trigger bounded reevaluation for previously discovered environments, including those still in `DISCOVERY` due to missing targets.
- [ ] Live read endpoint/source is known, permissioned and returns current image facts; timestamp/revision semantics checked.
- [ ] Base-image source supports reliable identity, new/changed release detection, and actual availability semantics.
- [ ] Equal-timestamp conflicts, source retries, unknown environment messages and poison-message policy have an approved handling strategy.

The purpose of this document is to keep the service honest about **what it knows**. All source-dependent details remain adaptable behind the source interfaces.
