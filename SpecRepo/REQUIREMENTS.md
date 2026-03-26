# Requirements

## Functional Requirements

### FR-1: Evaluate Assignment

- Actor: inference routing layer
- Trigger: an admitted inference request includes request identity and experiment or routing policy context
- The system must:
  - resolve the active published experiment version for the applicable routing configuration
  - deterministically assign the request to a treatment according to configured rules
  - return `variant_key` or `not_eligible` plus evaluation metadata sufficient for audit and replay
- Failure behavior:
  - if the experiment is unknown, inactive, or not applicable, return deterministic `not_eligible`

### FR-2: Persist Authoritative Decision Record

- Actor: inference routing layer
- Trigger: a routing decision is finalized for an admitted request
- The system must:
  - persist exactly one authoritative decision record per admitted request using a stable `decision_id`
  - bind that record to request identity, selected target, experiment metadata, snapshot identifiers, tenant identity, and decision timestamp
  - require downstream audit, chargeback, and execution representations to remain joinable to that record
- Failure behavior:
  - if the authoritative record cannot be persisted, the request must be rejected

### FR-3: Inference Routing Decision

- Actor: inference routing layer
- Trigger: a request arrives with resolved request identity and assignment result
- The system must:
  - resolve a versioned decision snapshot
  - apply admission control before any external call
  - determine the admissible target set using experiment constraints, tenant isolation rules, budget policy, and publish eligibility
  - select exactly one target from the admissible set according to routing policy
  - bind the final decision to explicit version identifiers
- Failure behavior:
  - if no valid snapshot or no admissible target exists, reject with deterministic errors

### FR-4: Execute Inference Through Platform Proxy

- Actor: inference serving layer
- Trigger: a routing decision selects an admitted target
- The system must:
  - execute inference through the platform-controlled serving layer in V1
  - dispatch execution traffic to either:
    - an internally hosted inference service
    - an external provider endpoint
  - treat internal and external execution targets as distinct target classes under the same routing decision contract
  - preserve the authoritative `decision_id`
  - capture terminal outcome information needed for audit and chargeback

### FR-4A: Reconcile Internal Inference Workloads

- Actor: control-plane deployment reconciler
- Trigger: a model version is promoted, an experiment rollout requires an internal serving target, or serving desired state changes
- The system must:
  - maintain explicit desired state for internally hosted inference workloads
  - persist desired and observed internal inference deployment state in CockroachDB as the authoritative serving-state store
  - reconcile that desired state into workload resources such as Kubernetes deployments, replica sets, services, or equivalent serving primitives
  - bind each deployed internal inference workload to:
    - target identifier
    - model version
    - artifact manifest URI
    - execution class (`cpu` or `gpu`)
    - replica policy
    - environment or namespace
  - record reconciliation progress and readiness state in CockroachDB before the workload becomes eligible for routing
- Failure behavior:
  - if the workload cannot be reconciled or does not become ready, the target must remain unroutable

### FR-4B: Expose Internal Inference Deployment APIs

- Actor: control-plane operator or automated promotion workflow
- Trigger: a caller needs to create, update, inspect, or retire an internal inference workload
- The system must expose:
  - `POST /api/v1/inference-deployments`
  - `PATCH /api/v1/inference-deployments/{deployment_id}`
  - `GET /api/v1/inference-deployments/{deployment_id}`
  - `POST /api/v1/inference-deployments/{deployment_id}:retire`
- The create or update path may cause pods to be created, replaced, or scaled by the deployment reconciler.

### FR-4C: Gate Routing On Internal Deployment Readiness

- Actor: routing layer
- Trigger: a routing policy references an internal inference target
- The system must:
  - consider an internal target admissible only when the corresponding deployment is in a ready state as recorded in CockroachDB
  - reject or fall back according to routing policy when an internal deployment is missing, unhealthy, or not yet ready
  - bind the selected target to the exact deployed model version intended by control-plane state

### FR-5: Publish Experiment and Routing Configuration

- Actor: control-plane operator via API
- Trigger: a draft routing policy or experiment configuration is promoted
- The system must:
  - validate targeting and routing rules
  - persist a new immutable published version
  - make that version available to snapshot construction and routing

### FR-5I: Support Experiment-Driven Internal Deployment Rollout

- Actor: control-plane operator or rollout automation
- Trigger: an experiment version or routing policy introduces an internal target that is not yet deployed at the required capacity or version
- The system must:
  - allow experiment and routing publication workflows to declare required internal deployment state
  - reconcile or verify that required internal inference workloads exist before the target becomes eligible for assignment
  - support rollout stages where an internal deployment exists but is not yet traffic-bearing until readiness and policy conditions are met

### FR-5E: Govern Experiment Lifecycle and Guardrails

- Actor: control-plane operator or automated experiment-governance workflow
- Trigger: an experiment is created, activated, paused, completed, or archived
- The system must:
  - support explicit experiment states:
    - `draft`
    - `validated`
    - `active`
    - `paused`
    - `completed`
    - `archived`
  - require every published experiment version to define:
    - assignment key
    - variant allocation
    - eligibility rules
    - success metrics
    - guardrail metrics
    - owner
    - start criteria
    - stop criteria
  - prevent a version from becoming `active` unless targeting, allocation, and metric definitions are valid
  - allow `paused` state to stop new exposure assignment without mutating prior version contents

### FR-5F: Emit Experiment Exposure Records

- Actor: inference routing layer
- Trigger: a request is evaluated against an active experiment
- The system must:
  - emit at most one immutable exposure record per admitted request per evaluated experiment
  - bind the exposure record to:
    - `decision_id`
    - experiment identifier
    - experiment version
    - variant key or `not_eligible`
    - assignment key
    - tenant identifier
    - exposure timestamp
  - emit exposure data even when the evaluated result is `not_eligible` if the system needs that outcome for experiment accounting

### FR-5G: Provide Experiment Analytics Inputs

- Actor: experiment analytics consumers
- Trigger: a published experiment requires analysis or governance review
- The system must:
  - make exposure, decision, execution outcome, audit, chargeback, and label data joinable by stable identifiers
  - define the minimum experiment-analysis input contract to include:
    - experiment version
    - variant key
    - assignment key
    - decision timestamp
    - selected target identifier
    - execution outcome
    - billable usage and cost where available
    - downstream label or outcome join key where available
  - record missing analysis inputs explicitly rather than silently omitting exposed traffic from experiment accounting

### FR-5A: Expose API-Managed Control-Plane Administration

- Actor: control-plane operator
- Trigger: an operator needs to manage routes, models, experiments, or published state
- The system must:
  - expose API endpoints for model registration, promotion, experiment creation and publication, route creation and update, routing-policy publication, decision inspection, projection inspection, and snapshot inspection
  - support operator workflows without requiring a first-party browser UI
  - keep operator actions auditable and version-bound under the same control-plane rules as other managed changes

### FR-5B: Expose Experiment Management APIs

- Actor: control-plane operator
- Trigger: an operator needs to create, modify, publish, or inspect an experiment
- The system must expose:
  - `POST /api/v1/experiments`
  - `PATCH /api/v1/experiments/{experiment_id}`
  - `POST /api/v1/experiments/{experiment_id}:publish`
  - `GET /api/v1/experiments/{experiment_id}`

### FR-5C: Expose Route Management APIs

- Actor: control-plane operator
- Trigger: an operator needs to create, update, or inspect an inference route mapping
- The system must expose:
  - `POST /api/v1/routes`
  - `PATCH /api/v1/routes/{route_id}`
  - `GET /api/v1/routes/{route_id}`

### FR-5H: Define Routing Policy Schema

- Actor: control-plane operator, routing layer, and replay consumers
- Trigger: a routing policy is created, updated, published, or replayed
- The system must define a routing-policy schema that includes, at minimum:
  - `policy_id`
  - `policy_version`
  - route or surface identifier
  - tenant scope
  - target set
  - target eligibility predicates
  - experiment bindings
  - fallback order
  - timeout policy
  - retry policy
  - budget or quota references
  - regional constraints
  - degradation behavior
  - publication metadata
- The system must treat policy versions as immutable and replayable.
- The system must reject policy publication if schema-required fields are missing or contradictory.

### FR-5D: Expose Decision and Projection Inspection APIs

- Actor: operator, SRE, or audit consumer
- Trigger: a caller needs to inspect decision or derived-record state for a known `decision_id`
- The system must expose:
  - `GET /api/v1/decisions/{decision_id}`
  - `GET /api/v1/decisions/{decision_id}/projections`
- The decision inspection endpoint must return the authoritative decision record and replay-relevant version identifiers.
- The projection inspection endpoint must return execution, audit, and chargeback projection status for that decision.

### FR-6: Multi-Region Snapshot Distribution

- Actor: snapshot distribution layer
- Trigger: published configuration or provider-state inputs change
- The system must:
  - distribute versioned snapshots to regional caches
  - record the snapshot version and region used for each routing decision
  - expose staleness metadata

### FR-7: Derive Chargeback and Audit Representations

- Actor: projection layers
- Trigger: an authoritative decision reaches terminal outcome
- The system must:
  - derive immutable, tenant-scoped, joinable chargeback and audit records keyed by `decision_id`
  - record failures explicitly rather than dropping records

### FR-7A: Derive Replayable Feedback Records

- Actor: feedback and replay consumers
- Trigger: an authoritative decision reaches later lifecycle stages or downstream truth arrives
- The system must define and persist immutable feedback-oriented records for:
  - execution outcomes
  - experiment exposures
  - downstream labels or truth bindings
  - replay requests and replay results
- Replay-oriented records must preserve the identifiers and version references required to reconstruct the evaluated routing context.

### FR-7B: Expose Replay APIs

- Actor: operator, SRE, audit consumer, or model-evaluation workflow
- Trigger: a caller needs to replay a prior decision or request identity against historical or current policy state
- The system must expose:
  - `POST /api/v1/replays`
  - `GET /api/v1/replays/{replay_id}`
- The replay request must allow, at minimum:
  - reference by `decision_id` or explicit request identity
  - replay mode against historical bound versions or current published versions
  - region selection when relevant
- The replay result must return:
  - replay input reference
  - version set used
  - replayed assignment and routing outcome
  - divergence from original decision when applicable

### FR-8: Build Training Rows For Repo-Owned Models

- Actor: training job
- Trigger: `ml/train.py` is invoked for a supported `--feature-group`
- The system must:
  - support `customer`, `customer_realtime`, `campaign`, and `advertiser`
  - resolve runtime training tables from the absorbed ML workflow configuration
  - use these default runtime training tables:
    - `customer` -> `iceberg.silver.customer_purchase_features_v1`
    - `customer_realtime` -> `iceberg.silver.customer_purchase_realtime_features_v1`
    - `campaign` -> `iceberg.silver.campaign_success_features_v1`
    - `advertiser` -> `iceberg.silver.advertiser_budget_features_v1`
  - preserve the file-path compatibility path used by existing tests

### FR-9: Train and Emit Model Artifacts

- Actor: training job
- Trigger: training rows are available
- The system must:
  - split rows chronologically
  - fit the current repository logistic-regression baseline
  - compute classifier metrics
  - write local dataset, model, metrics, and manifest artifacts
  - optionally publish those artifacts to MinIO-compatible object storage

### FR-10: Register Model Versions

- Actor: training job
- Trigger: `register_model=True`
- The system must:
  - create `iceberg.silver.ml_model_registry` if missing
  - insert a row containing feature group, label, feature-definition version, timestamps, artifact URIs, local paths, training row counts, and summary metrics
  - record the normal post-training status as `candidate` when required validation and test gates pass
  - avoid implicitly promoting a newly registered version to `production`

### FR-10B: Govern Repo-Owned Model Lifecycle

- Actor: control-plane operator, ML/platform engineer, or automated promotion workflow
- Trigger: a repo-owned model version is registered, promoted, rolled back, deprecated, or retired
- The system must support explicit model lifecycle states:
  - `candidate`
  - `validated`
  - `production`
  - `rollback_candidate`
  - `deprecated`
  - `retired`
- The system must:
  - require promotion into `production` to be explicit and auditable
  - preserve prior production references for rollback
  - prevent retired versions from being newly selected for routing or scoring
  - expose deprecation metadata and retirement reason
  - treat passing validation as sufficient for candidate publication, not automatic production activation
  - support an explicit exception-based promotion path when validation or tests do not pass, provided the override captures approver identity, reason, and risk acceptance metadata

### FR-10C: Expose Model Lifecycle APIs

- Actor: control-plane operator or automation
- Trigger: a caller needs to inspect or mutate lifecycle state for a repo-owned model version
- The system must expose:
  - `POST /api/v1/model-registry/{feature_group}/versions/{model_version}:promote`
  - `POST /api/v1/model-registry/{feature_group}/versions/{model_version}:deprecate`
  - `POST /api/v1/model-registry/{feature_group}/versions/{model_version}:retire`
  - `GET /api/v1/model-registry/{feature_group}/versions/{model_version}`
- The promote endpoint must support:
  - normal promotion from a validated candidate version
  - an exception-based promotion mode that records override metadata when required validation or tests did not pass
- Promotion of an internally hosted model version may trigger creation or update of an internal inference deployment before production routing activation succeeds.

### FR-10A: Expose Training Job APIs

- Actor: control-plane operator or automation
- Trigger: a caller needs to start or inspect repo-owned model training
- The system must expose:
  - `POST /api/v1/training-jobs`
  - `GET /api/v1/training-jobs/{training_job_id}`
- The create endpoint must accept, at minimum, the requested feature group and any explicit table or artifact overrides supported by the training workflow.
- The read endpoint must expose job status, validation/test outcomes, and any resulting artifact or registry references.
- When validation passes, the normal outcome is candidate publication only.
- When validation fails, the job must not publish through the normal candidate-first path, but the resulting version may still be eligible for explicit exception-based promotion through the lifecycle API.

### FR-11: Resolve Active Repo-Owned Model For Inference

- Actor: scoring library or inference API
- Trigger: a score request for a repo-owned model feature group is received
- The system must:
  - query `iceberg.silver.ml_model_registry`
  - select the latest manifest URI for that feature group ordered by `trained_at DESC`
  - load the manifest and referenced model from `s3://` or local disk

### FR-11A: Expose Model Registry Inspection APIs

- Actor: operator, automation, or inference-adjacent service
- Trigger: a caller needs to inspect current or historical registry state for a repo-owned model feature group
- The system must expose:
  - `GET /api/v1/model-registry/{feature_group}/latest`
  - `GET /api/v1/model-registry/{feature_group}/versions`

### FR-12: Expose ML Inference Endpoints

- Actor: FastAPI inference service
- Trigger: HTTP requests reach the ML inference API
- The system must expose:
  - `GET /health`
  - `GET /models/latest`
  - `POST /score/customer_purchase`
  - `POST /score/campaign_success`
  - `POST /score/advertiser_budget_expansion`

### FR-13: Score Customer Realtime With Hot Features

- Actor: scoring library or inference API
- Trigger: customer score request
- The system must:
  - load the latest `customer_realtime` model unless a manifest override is supplied
  - query offline context from:
    - `iceberg.silver.silver_customer_daily_metrics`
    - `iceberg.silver.silver_order_header`
    - `iceberg.silver.customer_realtime_features_v1_parity`
  - fetch the current Redis customer online feature record
  - build the scoring payload using:
    - `views_1h`
    - `views_24h`
    - `ad_clicks_24h`
    - `add_to_cart_24h`
    - `purchases_30d`
    - `avg_order_value_90d`
    - `days_since_last_purchase`
  - prefer Redis values over offline values when present
  - optionally write the score output to Redis

### FR-14: Score Campaign and Advertiser

- Actor: scoring library or inference API
- Trigger: campaign or advertiser score request
- The system must:
  - load the latest model for the requested feature group
  - query campaign offline features from `iceberg.silver.silver_campaign_daily_metrics`
  - query advertiser offline features from `iceberg.silver.silver_advertiser_daily_metrics`
  - score without requiring Redis in the current implementation

### FR-15: Maintain Customer Online Features In Redis

- Actor: Flink online-feature job
- Trigger: valid session events are consumed from Kafka topic `events.session_event`
- The system must:
  - decode events using the configured schema-registry URL
  - accept only `product_view`, `ad_click`, and `add_to_cart`
  - compute rolling distinct-count aggregates for:
    - `views_1h`
    - `views_24h`
    - `ad_clicks_24h`
    - `add_to_cart_24h`
  - write one Redis hash per customer using key pattern `features:customer:{customer_id}:v1`
  - set the key TTL to `86400` seconds

### FR-15A: Define Online Feature Platform Contract

- Actor: serving platform, online-feature job, and parity consumers
- Trigger: an online feature definition is introduced or revised
- The system must require each online feature definition to declare, at minimum:
  - feature name and version
  - owning service or team
  - entity type
  - entity key
  - key pattern
  - source streams or upstream dependencies
  - transformation or aggregation semantics
  - freshness expectation
  - TTL expectation
  - parity source or reconciliation strategy
  - rebuild strategy or recovery notes
- The system must treat the combination of feature name and version as the serving contract identity.

### FR-15B: Track Online Feature Freshness and Recovery State

- Actor: serving platform and operators
- Trigger: online feature records are written, inspected, rebuilt, or found stale
- The system must:
  - record freshness metadata sufficient to determine staleness for a feature record
  - distinguish between:
    - missing
    - fresh
    - stale
    - rebuilding
    - parity_failed
  - expose stale or missing online feature state to consumers and operators
  - define how rebuild or replay operations repopulate feature records without silently changing feature version identity

### FR-15C: Expose Online Feature Inspection APIs

- Actor: operator, SRE, or inference-adjacent service
- Trigger: a caller needs to inspect online feature definitions or current online feature record state
- The system must expose:
  - `GET /api/v1/online-features/definitions`
  - `GET /api/v1/online-features/{entity_type}/{entity_id}`
  - `POST /api/v1/online-features/{entity_type}/{entity_id}:rebuild`

### FR-16: Reconcile Online Feature Records

- Actor: serving-owned parity and reconciliation logic
- Trigger: a Redis customer online record is compared against the expected representation
- The system must:
  - derive the expected record from realtime parity input, customer feature row, and online feature definition metadata
  - normalize actual Redis payloads into typed values
  - compare field-level values and return mismatches rather than only a boolean result

### FR-16A: Expose Parity Inspection API

- Actor: operator, SRE, or serving engineer
- Trigger: a caller needs to inspect current customer realtime parity state
- The system must expose:
  - `GET /api/v1/parity/customer-online/{customer_id}`
- The endpoint must return the expected online record, the actual Redis record, and field-level mismatches.

### FR-16B: Persist Parity Results

- Actor: serving-owned reconciliation workflow
- Trigger: a parity comparison is executed
- The system must:
  - persist an immutable parity result containing:
    - feature definition identity
    - entity identifier
    - comparison timestamp
    - expected payload reference
    - actual payload reference
    - mismatch summary
    - pass/fail outcome
  - keep parity results joinable to online feature version and serving context

## Non-Functional Requirements

### Performance and Reliability

- p95 routing decision latency target remains under `300ms`
- routing-layer availability target remains `99.99%`
- published policy and experiment versions must be durable before activation
- operator management is API-only; no frontend availability target is required by this repo

### Deployment Placement

- if a dedicated GPU node pool is provisioned, only inference-serving workloads that execute model inference may tolerate GPU taints or select GPU nodes
- control-plane API, routing API, snapshot builder, projection workers, feedback/replay services, training job orchestration, registry inspection APIs, online feature services, Redis, PostgreSQL, CockroachDB, Kafka, and other supporting infrastructure must schedule onto the default non-GPU node pool
- the existence of a GPU node pool must not broaden placement for general compute workloads in this repo

### Shared Resource Reuse

- when prerequisite resources already exist in `../example-data-pipeline-w-ml`, the default deployment model is to extend those resources rather than stand up duplicates
- this applies especially to:
  - PostgreSQL
  - MinIO-compatible object storage
  - Kafka
  - Kafka Connect
  - Iceberg
  - Schema Registry
  - dbt
- CockroachDB is the explicit exception to generic relational-store reuse: it is the authoritative serving-state database for internal inference deployment desired state, observed state, reconciliation progress, and readiness-gated controls
- new dedicated instances should be introduced only when there is a clear requirement for isolation, incompatible lifecycle, capacity separation, or security boundary separation
- reused shared resources must still provide:
  - explicit namespace, schema, bucket, topic, connector, catalog, and job naming for this repo's assets
  - auditable ownership of objects created by this repo
  - compatibility with the existing upstream platform's operational model

### Runtime Model For Absorbed ML Workflow

- the current implementation is Python 3.11 oriented
- training and inference use repository-local code rather than heavyweight external ML frameworks
- training can run in a container via `config/ml-training/Dockerfile` and `train-models.sh`
- object storage is MinIO-compatible and addressed through S3 APIs
- registry I/O is performed through Trino
- CockroachDB stores authoritative internal inference deployment desired state, observed state, reconciliation progress, and readiness-gated serving controls
- Redis is required for the customer hot-feature path

### Tests

- unit tests cover label logic, metrics, and point-in-time feature assembly expectations
- integration tests cover local artifact emission and customer online-record reconciliation
- some absorbed tests are currently blocked by the missing `offline_feature_defs.yaml`
