# Problem Framing

## One-Sentence Mission

We are building a centralized routing and policy control plane that deterministically selects and serves models for inference requests across tenants, and we now also own the training, model-registry, inference, and hot-feature-serving workflow for repo-owned models used by that platform.

## Users / Actors

Primary:

- internal product teams consuming routed inference
- enterprise customers receiving routed model behavior
- ML/platform engineers training and promoting repo-owned models

Secondary:

- platform management
- SRE
- finance and chargeback consumers
- upstream data platform teams publishing offline features and events

## Scope Boundaries

### In Scope

- routing and policy control plane for inference requests
- request-time model selection based on policy, availability, latency, cost, and experiment state
- budget and authorization enforcement
- versioned routing policy publication and immutable capture of referenced GrowthBook experiment bindings
- immutable routing, audit, and chargeback records
- feedback and data contracts for decisions, outcomes, exposures, labels, and replay
- experiment analytics, governance, and guardrail evaluation for routing-controlled experiments
- GrowthBook as the experimentation control plane for assignment, targeting, exposure tracking, and evaluation
- API-managed operator workflows for control-plane administration
- ML-owned model training for the current baseline classifiers
- model artifact publication to MinIO-compatible object storage
- model registry writes and reads through `iceberg.silver.ml_model_registry`
- model lifecycle governance for candidate, production, rollback, deprecation, and retirement state
- inference APIs for repo-owned customer, campaign, and advertiser models
- execution routing across both internally hosted inference services and externally hosted provider endpoints
- deployment and reconciliation of internally hosted inference workloads, including API-driven creation of serving pods when required by promotion or experiment rollout
- CockroachDB-backed authoritative serving state for internal inference deployments, reconciliation progress, and readiness-gated controls
- Redis-backed online feature serving for the customer realtime path
- serving-owned offline/online parity and reconciliation for that online feature path
- broader online feature platform semantics for feature definitions, freshness, rebuild, and serving contracts
- integration with shared prerequisite platform resources from `../example-data-pipeline-w-ml` where reuse is viable

### Out of Scope

- a first-party browser UI for platform management
- upstream offline feature-table generation and backfills
- ownership of the data platform's generic Iceberg datasets outside ML registry writes
- Kafka topic production
- schema-registry administration
- GPU scheduling and generic infrastructure provisioning
- product-specific quality-threshold definition

## Ownership Boundary Versus Data Platform

This repo now owns:

- model training
- model registry writes and reads
- inference APIs
- experimentation and rollout logic
- Redis-backed online feature serving
- offline/online feature parity checks owned by serving

The upstream data repo remains an external dependency only. It publishes offline feature tables and upstream events, but those datasets and pipelines are not re-owned here.

Shared prerequisite resources from `../example-data-pipeline-w-ml` may be reused and extended rather than duplicated, especially:

- PostgreSQL
- MinIO-compatible object storage
- Kafka
- Kafka Connect
- Iceberg
- Schema Registry
- dbt

This reuse is mandatory for deployment in DigitalOcean Kubernetes. This repo must consume the existing PostgreSQL, MinIO, Kafka, and Iceberg platform services rather than introducing duplicate services here.

## Explicit Upstream Dependencies

The absorbed training workflow depends on:

- `iceberg.silver.customer_purchase_features_v1`
- `iceberg.silver.customer_purchase_realtime_features_v1`
- `iceberg.silver.campaign_success_features_v1`
- `iceberg.silver.advertiser_budget_features_v1`

The current scoring code additionally depends on:

- `iceberg.silver.silver_customer_daily_metrics`
- `iceberg.silver.silver_order_header`
- `iceberg.silver.customer_realtime_features_v1_parity`
- `iceberg.silver.silver_campaign_daily_metrics`
- `iceberg.silver.silver_advertiser_daily_metrics`

## Success Definition

The system is successful when:

- routing decisions remain deterministic, version-bound, and auditable
- admitted requests remain attributable and budget-safe
- decision, exposure, outcome, audit, chargeback, and label data remain joinable under explicit contracts
- experiment governance can distinguish draft, running, paused, and completed states with guardrail-aware analysis inputs
- GrowthBook-driven experiments use deterministic server-side assignment with user-level stickiness and warehouse-SQL metric definitions
- routing policy can be reasoned about as an explicit schema rather than ad hoc operator intent
- repo-owned model training can materialize artifacts, publish them, and register them
- repo-owned models can move through an explicit lifecycle rather than implicit latest-row selection only
- inference can resolve the current repo-owned model version from the registry
- internal inference targets can be deployed, become ready, and receive traffic under explicit control-plane and experiment-rollout rules
- experiment evaluation uses GrowthBook-compatible statistics over Iceberg/Trino-backed metrics, including CUPED, Bayesian inference, and sequential testing
- customer realtime scoring can merge offline context with Redis-backed hot features
- campaign and advertiser scoring can execute from offline features
- serving parity checks can detect mismatches between expected and actual Redis online records
- online feature definitions can express entity keys, freshness, TTL, rebuild expectations, and parity obligations

## Explicit Open Questions

- `config/features/offline_feature_defs.yaml` is referenced by the absorbed ML code but missing
- `tools/demo_realtime_scoring.py` references `score_entities(...)`, which is absent
- hard-coded data-platform hostnames still exist in some absorbed runtime defaults
- the current rollout mechanism for absorbed ML models is effectively latest-manifest selection by `trained_at`; no separate model-release state machine exists in code
- the current absorbed training flow does not yet define a stable dataset-version contract separate from feature table names and timestamps
- the intended normal promotion policy is candidate-first after successful validation, with explicit production promotion later; manual override promotion remains possible by exception when validation does not pass
