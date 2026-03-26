# Consistency Model

## Consistency Goals

### Workflow: Routing Decision and Admission

- Required guarantee:
  - for a given request identity and snapshot, routing evaluation is deterministic and produces at most one committed authoritative decision record per admitted request
- Failure tolerance:
  - fail closed if authentication, authorization, budget validation, or valid snapshot resolution cannot be proven

### Workflow: Internal Inference Deployment Reconciliation

- Required guarantee:
  - internal serving desired state and observed deployment state are durably persisted in CockroachDB and reconciled into explicit workload state before routing treats the target as ready
- Failure tolerance:
  - deployment reconciliation failure leaves the target unroutable or fallback-only according to policy

### Workflow: Configuration Publish

- Required guarantee:
  - published policy and experiment versions are immutable, monotonically versioned, and durably persisted before routing uses them

### Workflow: Experiment Exposure and Analytics Input

- Required guarantee:
  - exposure records bind to the exact experiment version and decision identifiers used at assignment time
- Failure tolerance:
  - if an exposure record cannot be persisted, the failure must be explicit for reconciliation; exposed traffic must not silently disappear from analysis inputs

### Workflow: Replay

- Required guarantee:
  - replay against historical mode uses the exact bound versions recorded on the original decision when available
  - replay against current mode records the current version set used for evaluation
- Failure tolerance:
  - replay failure must be explicit and must preserve the replay request record

### Workflow: Training Artifact Emission

- Required guarantee:
  - one training invocation emits a self-consistent local artifact set for one model version
- Commit point:
  - the manifest has been written after dataset, model, and metrics files exist

### Workflow: Artifact Publication and Registry Write

- Required guarantee:
  - when publication and registration are enabled, the registry row must point at the artifact URIs produced by that training run
- Commit points:
  - object publication commits when the final manifest URI exists
  - registry write commits when the Trino `INSERT` completes
- Non-guarantee:
  - no single transaction spans local artifact emission, object-store upload, and registry insert

### Workflow: Active Model Resolution

- Required guarantee:
  - repo-owned model inference resolves exactly one latest manifest URI per feature group using `ORDER BY trained_at DESC LIMIT 1`
- Staleness tolerance:
  - eventual visibility through Trino is acceptable

### Workflow: Model Lifecycle Governance

- Required guarantee:
  - lifecycle transitions for repo-owned models are append-only, explicit, and auditable
- Failure tolerance:
  - promotion, deprecation, or retirement must fail closed if the target version identity or required metadata cannot be proven
  - validation failure blocks the normal candidate-first path but does not remove the availability of an explicit override promotion workflow when the caller supplies required approval metadata

### Workflow: Customer Hot Feature Serving

- Required guarantee:
  - each processed event updates the per-customer Redis hash and TTL according to the current keyed state
- Staleness tolerance:
  - Redis is eventually updated relative to Kafka arrival
- Non-guarantee:
  - the implementation does not define exactly-once Redis semantics

### Workflow: Online Feature Parity and Rebuild

- Required guarantee:
  - parity results are computed against explicit feature-version identity and preserved immutably
  - rebuild actions must preserve or explicitly change feature-version identity rather than silently mutating contract meaning
- Failure tolerance:
  - parity or rebuild failure must be surfaced as explicit state, not inferred from missing data alone

## Read Semantics

- strong reads are required for authoritative decision lookup, published version lookup, and decision replay
- strong reads are required from CockroachDB for internal inference deployment readiness, reconciliation status, and desired-state lookup used in routing eligibility
- stale-tolerant reads are allowed for regional snapshot caches, derived projections, and Redis hot features
- repo-owned model inference reads are manifest-driven and strong relative to the manifest URI returned by the registry query

## Ordering Guarantees

- routing records are ordered by immutable version identifiers and `decision_id`
- published configurations are monotonically versioned
- experiment exposures are ordered by exposure timestamp and bind to immutable experiment version identifiers
- internal deployment readiness must be established before the corresponding target is treated as eligible for routing
- training rows are ordered chronologically before splitting
- active-model selection for absorbed ML models is ordered only by `trained_at DESC`
- customer hot-feature windows are ordered by retained event timestamps in keyed state

## Concurrency

- duplicate admissions collapse via the authoritative decision record rules
- concurrent publishes produce distinct immutable versions rather than in-place overwrite
- concurrent training runs for the same feature group are allowed by current code and may create multiple registry rows; latest `trained_at` wins for current-model lookup
- concurrent replay requests are allowed and must remain distinct by replay identifier
- concurrent lifecycle transitions must resolve by explicit append-only state transitions rather than in-place overwrites
- concurrent deployment updates for the same internal target must converge on one latest desired state for that target and model version

## Explicit Non-Guarantees

- no globally fresh snapshot guarantee across all regions at once
- no synchronous visibility guarantee for derived chargeback or audit records
- no transactional rollback across artifact publication and registry insert
- no exactly-once contract for Redis hot-feature writes
- no guarantee that latest-`trained_at` model selection alone is sufficient long term for production governance; the lifecycle contract must govern when present
