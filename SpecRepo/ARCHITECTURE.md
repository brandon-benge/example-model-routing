# Architecture

## Overview

The governed system remains a centralized model routing and policy control plane with control-plane/data-plane separation, immutable versioned configuration, and deterministic routing against explicit snapshots.

The absorbed ML workflow is an additional subsystem inside that architecture, not a replacement for it. It adds:

- batch training through `ml/train.py`
- model artifact persistence in local storage plus MinIO-compatible object storage
- model registry reads and writes through Trino into Iceberg
- FastAPI-based inference for repo-owned models
- Redis-backed customer hot feature serving
- a Flink job that aggregates session events into Redis customer feature hashes
- feedback and replay contracts that connect routed decisions to downstream audit, chargeback, labels, and experiment analysis inputs
- model lifecycle governance for repo-owned trained versions
- broader online feature platform semantics beyond a single Redis hash path
- a required integration stance that reuses prerequisite shared platform resources from `../example-data-pipeline-w-ml`
- CockroachDB as the authoritative serving-state store for internal inference deployment reconciliation and readiness-gated controls
- GrowthBook as the experimentation control plane, using server-side SDK evaluation and warehouse-SQL metrics

## Components

### Component: Control Plane API

- Responsibility:
  - CRUD and publish workflows for routing policy, endpoint policy, GrowthBook experiment references, model catalog metadata, and operator audit
  - API-only management for operator workflows; no first-party browser UI is assumed
- Canonical operator-facing endpoints:
  - `POST /api/v1/model-targets`
  - `POST /api/v1/model-targets/{model_target_id}:promote`
  - `POST /api/v1/routes`
  - `PATCH /api/v1/routes/{route_id}`
  - `POST /api/v1/routing-policies/{policy_id}:publish`
  - `GET /api/v1/snapshots/{region}/current`
  - `GET /api/v1/decisions/{decision_id}`
  - `GET /api/v1/decisions/{decision_id}/projections`
  - `POST /api/v1/replays`
  - `GET /api/v1/replays/{replay_id}`

### Component: GrowthBook Experimentation Control Plane

- Responsibility:
  - store experiment definitions, targeting rules, allocation settings, and analysis configuration for governed experiments
  - provide deterministic server-side SDK evaluation with user-level stickiness
  - support dynamic traffic allocation, lifecycle state changes, and kill-switch behavior without this repo storing a duplicate experiment-definition payload
  - evaluate backend and ML experiments covering model routing, prompt variants, and feature-pipeline variants
  - serve as the only experiment-management interface; this repo does not expose a parallel experiment CRUD or lifecycle API

### Component: Snapshot Builder and Distributor

- Responsibility:
  - assemble and distribute immutable regional snapshots used by routing
  - embed captured GrowthBook experiment references and rule bindings alongside routing-policy state

### Component: Routing API

- Responsibility:
  - authenticate requests, evaluate GrowthBook experiment assignment through the server-side SDK, resolve snapshots, enforce admission controls, persist authoritative decisions, and route requests
  - use GrowthBook experiment identifiers and captured rule bindings from snapshots rather than a duplicated local experiment-definition object

### Component: Platform Proxy / Execution Gateway

- Responsibility:
  - execute V1 inference through the platform-controlled path and emit execution outcomes
  - route execution traffic to either internally hosted inference services or external provider endpoints based on the selected target
- Deployment rule:
  - this is the only first-party service class in this repo permitted to target a dedicated GPU node pool when one exists

### Component: Internal Inference Deployment Reconciler

- Responsibility:
  - persist and read authoritative desired and observed serving state in CockroachDB
  - translate desired internal inference deployment state into actual serving workloads
  - create, update, scale, and retire pods or equivalent serving resources for internal targets
  - publish readiness and health state consumed by routing and promotion workflows

### Component: Projection Workers

- Responsibility:
  - derive exposure, execution, audit, and chargeback records keyed by `decision_id`

### Component: Feedback Contract and Replay Layer

- Responsibility:
  - define and persist joinable records for exposure, execution outcome, labels, and replay
  - support replay against historical or current version sets
  - preserve divergence analysis between original and replayed outcomes

### Component: Training CLI

- Implementation:
  - `ml/train.py`
- Responsibility:
  - read upstream feature rows, split chronologically, train the current logistic-regression baselines, write artifacts, and optionally publish/register them

### Component: Artifact Store Adapter

- Implementation:
  - `ml/artifact_store.py`
- Responsibility:
  - manage MinIO-compatible artifact upload and download

### Component: Model Registry Adapter

- Implementation:
  - `ml/trino_utils.py`
  - registry SQL in `ml/train.py` and `ml/inference.py`
- Responsibility:
  - write registry rows and resolve latest manifest URIs for repo-owned models

### Component: Repo-Owned Model Inference API

- Implementation:
  - `ml/inference_api.py`
  - `ml/inference.py`
  - `ml/scoring.py`
- Responsibility:
  - expose model-health and scoring endpoints
  - load the current model manifest and model
  - query offline context
  - merge Redis customer hot features where applicable
  - optionally persist score outputs to Redis
  - act as an internal inference target class that may be selected by the execution gateway
- Canonical endpoints:
  - `GET /health`
  - `GET /models/latest`
  - `POST /score/customer_purchase`
  - `POST /score/campaign_success`
  - `POST /score/advertiser_budget_expansion`

### Component: Training Job API Surface

- Responsibility:
  - expose API-driven initiation and inspection of repo-owned model training
- Canonical endpoints:
  - `POST /api/v1/training-jobs`
  - `GET /api/v1/training-jobs/{training_job_id}`

### Component: Model Registry Inspection API Surface

- Responsibility:
  - expose API-driven inspection of current and historical repo-owned model registry state
- Canonical endpoints:
  - `GET /api/v1/model-registry/{feature_group}/latest`
  - `GET /api/v1/model-registry/{feature_group}/versions`

### Component: Model Lifecycle Governance API Surface

- Responsibility:
  - expose explicit lifecycle governance for repo-owned model versions
- Canonical endpoints:
  - `POST /api/v1/model-registry/{feature_group}/versions/{model_version}:promote`
  - `POST /api/v1/model-registry/{feature_group}/versions/{model_version}:deprecate`
  - `POST /api/v1/model-registry/{feature_group}/versions/{model_version}:retire`
  - `GET /api/v1/model-registry/{feature_group}/versions/{model_version}`
- Governance rule:
  - normal flow promotes validated candidates only
  - exception flow permits explicit override promotion with auditable approval metadata when validation did not pass

### Component: Hot Feature Aggregator

- Implementation:
  - `flink/jobs/online_features_to_redis.py`
  - `config/features/online_feature_defs.yaml`
- Responsibility:
  - consume session events, compute rolling customer realtime aggregates, and write Redis hot-feature hashes

### Component: Online Feature Definition and Inspection API Surface

- Responsibility:
  - expose the authoritative online feature serving contract, freshness state, and rebuild controls
- Canonical endpoints:
  - `GET /api/v1/online-features/definitions`
  - `GET /api/v1/online-features/{entity_type}/{entity_id}`
  - `POST /api/v1/online-features/{entity_type}/{entity_id}:rebuild`

### Component: Redis Store

- Responsibility:
  - hold customer hot-feature hashes and optional score-output records

### Component: Parity Inspection API Surface

- Responsibility:
  - expose serving-owned parity inspection for customer realtime online records
- Canonical endpoint:
  - `GET /api/v1/parity/customer-online/{customer_id}`

### Component: Experiment Analytics and Governance Layer

- Responsibility:
  - consume experiment exposure, outcome, cost, and label inputs
  - resolve metric SQL over Iceberg/Trino-backed warehouse tables
  - execute GrowthBook statistical evaluation including CUPED, Bayesian inference, and sequential testing
  - support pause, completion, and kill-switch decisions without this repo mutating a duplicate experiment-definition payload

## Main Flows

### Primary Flow: Routed Inference

1. Caller sends an authenticated inference request.
2. Routing API evaluates GrowthBook assignment through the server-side SDK using sticky `user_id` bucketing and resolves the current regional snapshot.
3. Routing API applies authorization, budget, and target eligibility checks.
4. Routing API persists the authoritative decision record and mandatory exposure record.
5. Platform Proxy executes the selected target path, dispatching either to an internal inference service or to an external provider.
6. Projection workers derive execution, audit, and chargeback representations.

### Added Flow: Training and Model Registration

1. Operator or job invokes `ml/train.py` with a repo-owned model feature group.
2. Training reads upstream feature rows through Trino or compatibility file inputs.
3. Training writes dataset, model, metrics, and manifest artifacts locally.
4. Training optionally uploads those artifacts to MinIO-compatible storage.
5. Training optionally inserts a registry row into `iceberg.silver.ml_model_registry`.

### Added Flow: Internal Inference Deployment

1. A promotion or experiment/routing change declares required internal serving state.
2. The deployment reconciler creates or updates the internal inference workload.
3. Pods may be created or replaced to satisfy the requested model version and execution class.
4. Readiness is reported back into control-plane state.
5. Routing may only send traffic once the deployment is ready.

### Added Flow: Repo-Owned Model Inference

1. Caller hits one of the FastAPI score endpoints.
2. The service resolves the latest manifest URI for the relevant feature group.
3. The service loads the manifest and model.
4. The scoring library queries upstream offline context.
5. Customer scoring also fetches current Redis hot features and merges them with offline values.
6. The service returns the score plus supporting feature context and artifact metadata.

### Added Flow: Customer Hot Features

1. The Flink job consumes `events.session_event`.
2. It filters to allowed event types from the online feature definition.
3. It maintains customer-local event state.
4. It computes rolling aggregates and writes the Redis hot-feature hash with TTL.

### Added Flow: Feedback and Replay

1. A routed request emits an authoritative decision and experiment exposure data.
2. Execution produces terminal outcome information.
3. Audit, chargeback, and downstream label inputs remain joinable to that decision.
4. A replay request can evaluate the same request identity against historical or current version sets, including the same captured GrowthBook experiment reference and sticky assignment inputs.
5. Replay results preserve divergence from the original outcome when applicable.

## Ownership Boundaries

### Owned Here

- routing and policy control plane
- model training
- model registry writes and reads
- inference APIs
- experimentation and rollout logic
- Redis-backed online feature serving
- serving-owned parity and reconciliation checks

### External Dependencies

- upstream offline feature table publication
- upstream daily metrics and order tables used during scoring
- Kafka topic production
- schema registry
- Trino and Iceberg infrastructure
- object storage infrastructure
- Redis infrastructure

### Shared Resource Reuse Policy

- required posture:
  - run on Kubernetes in DigitalOcean and extend shared prerequisite resources from `../example-data-pipeline-w-ml` instead of duplicating them
- candidate shared resources:
  - PostgreSQL
  - MinIO-compatible object storage
  - Kafka
  - Kafka Connect
  - Iceberg
  - Schema Registry
  - dbt
- explicit exception:
  - CockroachDB remains the authoritative store for internal inference deployment desired state, observed state, reconciliation progress, and readiness-gated controls even when shared PostgreSQL is reused for other metadata
- required safeguards:
  - separate schemas, buckets, topics, connector names, Iceberg namespaces, and dbt models for this repo's assets
  - no ownership ambiguity over upstream platform-managed assets
  - no accidental coupling that prevents independent evolution of this repo's APIs and specs

## Open Architectural Questions

- what repository-owned service names should replace the remaining data-platform-specific defaults
- whether the absorbed model rollout should remain latest-manifest-by-`trained_at` or move under the fuller control-plane promotion model
- whether `customer` training remains a real runtime path or only compatibility for tests
